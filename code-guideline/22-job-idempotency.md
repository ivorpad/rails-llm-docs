# Job Idempotency Best Practices

## Core Principle
Jobs will be retried and must be idempotent. Design every job to be safely run multiple times without causing duplicate effects.

## Understanding Idempotency

### Why Jobs Need to Be Idempotent
- Network failures can cause retries
- Worker crashes trigger re-execution
- Sidekiq's at-least-once delivery guarantee
- Manual retries for debugging

## Idempotency Patterns

### Check Before Acting
```ruby
class ProcessPaymentJob
  include Sidekiq::Job
  
  def perform(payment_id)
    payment = Payment.find(payment_id)
    
    # Check if already processed
    return if payment.processed?
    
    # Use database lock to prevent race conditions
    payment.with_lock do
      # Double-check after acquiring lock
      return if payment.reload.processed?
      
      # Process the payment
      charge = PaymentGateway.charge(
        amount: payment.amount,
        token: payment.token
      )
      
      # Mark as processed with transaction ID
      payment.update!(
        processed: true,
        gateway_transaction_id: charge.transaction_id,
        processed_at: Time.current
      )
    end
  rescue PaymentGateway::DuplicateChargeError => e
    # Payment was already charged, mark as processed
    payment.update!(
      processed: true,
      gateway_transaction_id: e.transaction_id
    )
  end
end
```

### Idempotency Keys
```ruby
class CreateOrderJob
  include Sidekiq::Job
  
  def perform(order_params, idempotency_key)
    # Check if this operation was already performed
    existing = IdempotencyRecord.find_by(key: idempotency_key)
    return existing.result if existing
    
    # Perform the operation
    order = nil
    ActiveRecord::Base.transaction do
      order = Order.create!(order_params)
      
      # Record the idempotency key
      IdempotencyRecord.create!(
        key: idempotency_key,
        result: order.to_json,
        created_at: Time.current
      )
    end
    
    # Send notifications (also idempotent)
    SendOrderNotificationJob.perform_async(order.id)
    
    order
  rescue ActiveRecord::RecordNotUnique
    # Key already exists, return existing result
    IdempotencyRecord.find_by!(key: idempotency_key).result
  end
end

# Idempotency record model
class IdempotencyRecord < ApplicationRecord
  # Table: idempotency_records
  # - key: string, unique index
  # - result: jsonb
  # - created_at: timestamp
  
  # Clean up old records periodically
  scope :expired, -> { where('created_at < ?', 7.days.ago) }
end
```

### Using Unique Jobs
```ruby
# Gemfile
gem 'sidekiq-unique-jobs'

class SyncUserDataJob
  include Sidekiq::Job
  
  sidekiq_options(
    unique: :until_executed,
    unique_args: ->(args) { [args.first] }  # Unique per user_id
  )
  
  def perform(user_id)
    user = User.find(user_id)
    
    # Sync will only run once per user_id until completed
    ExternalApi.sync_user(user)
    
    user.update!(last_synced_at: Time.current)
  end
end
```

## Database Constraints for Idempotency

### Unique Constraints
```ruby
# Migration
class AddUniqueConstraintToSubscriptions < ActiveRecord::Migration[7.0]
  def change
    # Only one active subscription per user
    add_index :subscriptions, 
              [:user_id, :status], 
              unique: true,
              where: "status = 'active'",
              name: 'idx_one_active_subscription_per_user'
  end
end

class CreateSubscriptionJob
  include Sidekiq::Job
  
  def perform(user_id, plan_id)
    user = User.find(user_id)
    plan = Plan.find(plan_id)
    
    Subscription.create!(
      user: user,
      plan: plan,
      status: 'active',
      starts_at: Time.current
    )
  rescue ActiveRecord::RecordNotUnique
    # Subscription already exists, that's fine
    Rails.logger.info "Active subscription already exists for user #{user_id}"
  end
end
```

### Upsert Pattern
```ruby
class UpdateInventoryJob
  include Sidekiq::Job
  
  def perform(product_id, quantity_change)
    # Upsert ensures idempotency
    InventoryAdjustment.upsert(
      {
        product_id: product_id,
        adjustment_id: "#{product_id}-#{Date.current}",
        quantity: quantity_change,
        date: Date.current
      },
      unique_by: :adjustment_id
    )
    
    # Recalculate total inventory
    product = Product.find(product_id)
    total = InventoryAdjustment.where(product_id: product_id).sum(:quantity)
    product.update!(quantity: total)
  end
end
```

## External API Idempotency

### Handle Duplicate API Calls
```ruby
class SendEmailJob
  include Sidekiq::Job
  
  def perform(email_id)
    email = Email.find(email_id)
    
    return if email.sent?
    
    # Many email services support idempotency keys
    response = EmailService.send_email(
      to: email.recipient,
      subject: email.subject,
      body: email.body,
      idempotency_key: "email-#{email.id}"
    )
    
    email.update!(
      sent: true,
      sent_at: Time.current,
      message_id: response.message_id
    )
  rescue EmailService::AlreadySentError => e
    # Email was already sent, update our records
    email.update!(
      sent: true,
      sent_at: e.sent_at,
      message_id: e.message_id
    )
  end
end
```

### Webhook Processing
```ruby
class ProcessWebhookJob
  include Sidekiq::Job
  
  def perform(webhook_id)
    webhook = Webhook.find(webhook_id)
    
    # Skip if already processed
    return if webhook.processed?
    
    # Use event_id for idempotency
    existing = WebhookEvent.find_by(
      event_id: webhook.event_id,
      source: webhook.source
    )
    
    if existing
      webhook.update!(processed: true, duplicate_of: existing.id)
      return
    end
    
    # Process the webhook
    ActiveRecord::Base.transaction do
      event = WebhookEvent.create!(
        event_id: webhook.event_id,
        source: webhook.source,
        data: webhook.payload
      )
      
      # Process based on event type
      case webhook.event_type
      when 'payment.succeeded'
        ProcessPaymentWebhook.new.call(event)
      when 'subscription.cancelled'
        ProcessCancellationWebhook.new.call(event)
      end
      
      webhook.update!(processed: true)
    end
  end
end
```

## Time-Based Idempotency

### Daily Jobs
```ruby
class GenerateDailyReportJob
  include Sidekiq::Job
  
  def perform(date_string = nil)
    date = date_string ? Date.parse(date_string) : Date.current
    
    # Check if report already exists
    existing = DailyReport.find_by(date: date)
    return if existing && existing.completed?
    
    # Create or find report
    report = DailyReport.find_or_create_by(date: date) do |r|
      r.status = 'pending'
    end
    
    # Skip if another job is processing
    return unless report.pending?
    
    # Mark as processing with atomic update
    updated = DailyReport.where(
      id: report.id,
      status: 'pending'
    ).update_all(
      status: 'processing',
      started_at: Time.current
    )
    
    return if updated == 0  # Another job got it first
    
    begin
      # Generate report
      data = ReportGenerator.new.generate_for_date(date)
      
      report.update!(
        status: 'completed',
        data: data,
        completed_at: Time.current
      )
    rescue => e
      report.update!(
        status: 'failed',
        error_message: e.message
      )
      raise
    end
  end
end
```

## Testing Idempotency

### RSpec Tests
```ruby
RSpec.describe ProcessPaymentJob do
  let(:payment) { create(:payment, amount: 100) }
  
  it "is idempotent" do
    expect(PaymentGateway).to receive(:charge).once.and_return(
      double(transaction_id: 'txn_123')
    )
    
    # Run job multiple times
    3.times { described_class.new.perform(payment.id) }
    
    payment.reload
    expect(payment).to be_processed
    expect(payment.gateway_transaction_id).to eq('txn_123')
  end
  
  it "handles concurrent execution" do
    # Simulate concurrent execution
    threads = 3.times.map do
      Thread.new do
        described_class.new.perform(payment.id)
      end
    end
    
    threads.each(&:join)
    
    payment.reload
    expect(payment).to be_processed
    
    # Should only have one charge
    expect(Payment.where(id: payment.id, processed: true).count).to eq(1)
  end
  
  it "handles partial failure" do
    # First attempt fails after charging
    allow(PaymentGateway).to receive(:charge).and_return(
      double(transaction_id: 'txn_123')
    )
    allow_any_instance_of(Payment).to receive(:update!).and_raise(StandardError)
    
    expect { described_class.new.perform(payment.id) }.to raise_error(StandardError)
    
    # Second attempt should handle already-charged state
    allow_any_instance_of(Payment).to receive(:update!).and_call_original
    allow(PaymentGateway).to receive(:charge).and_raise(
      PaymentGateway::DuplicateChargeError.new(transaction_id: 'txn_123')
    )
    
    described_class.new.perform(payment.id)
    
    payment.reload
    expect(payment).to be_processed
    expect(payment.gateway_transaction_id).to eq('txn_123')
  end
end
```

## Cleanup and Maintenance

### Periodic Cleanup Job
```ruby
class CleanupIdempotencyRecordsJob
  include Sidekiq::Job
  
  sidekiq_options queue: 'maintenance'
  
  def perform
    # Clean up old idempotency records
    IdempotencyRecord.expired.delete_all
    
    # Clean up old webhook events
    WebhookEvent.where('created_at < ?', 30.days.ago).delete_all
    
    # Clean up completed daily reports
    DailyReport.where(
      status: 'completed',
      date: ...90.days.ago
    ).delete_all
  end
end

# Schedule to run daily
# Sidekiq::Cron::Job.create(
#   name: 'Cleanup idempotency records',
#   cron: '0 3 * * *',
#   class: 'CleanupIdempotencyRecordsJob'
# )
```

## Best Practices Summary

1. **Always check if work was already done**
2. **Use database constraints for uniqueness**
3. **Implement idempotency keys for critical operations**
4. **Handle duplicate API calls gracefully**
5. **Use locks to prevent race conditions**
6. **Test idempotency explicitly**
7. **Clean up idempotency records periodically**
8. **Log but don't fail on duplicate operations**