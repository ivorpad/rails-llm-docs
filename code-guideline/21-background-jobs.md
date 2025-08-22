# Background Jobs Best Practices

## When to Use Background Jobs

### Good Reasons for Background Jobs
1. **Network calls to third parties** - APIs can be slow or flaky
2. **Heavy computation** - Don't block web requests
3. **Batch processing** - Process large datasets
4. **Scheduled tasks** - Periodic cleanup, reports
5. **Email sending** - Never send email synchronously

### When NOT to Use Background Jobs
```ruby
# BAD - Unnecessary background job
class UpdateUserJob < ApplicationJob
  def perform(user_id, name)
    User.find(user_id).update!(name: name)
  end
end

# GOOD - Just do it synchronously
user.update!(name: name)
```

## Sidekiq Best Practices

### Use Sidekiq Directly, Not ActiveJob
```ruby
# BAD - ActiveJob adds unnecessary abstraction
class ProcessOrderJob < ApplicationJob
  queue_as :default
  
  def perform(order_id)
    # Job logic
  end
end

# GOOD - Direct Sidekiq usage
class ProcessOrderJob
  include Sidekiq::Job
  
  sidekiq_options queue: 'critical', retry: 3
  
  def perform(order_id)
    order = Order.find(order_id)
    ProcessOrder.new.call(order: order)
  rescue ActiveRecord::RecordNotFound
    # Order was deleted, that's ok
  end
end
```

### Configure Queues by Priority
```ruby
# config/sidekiq.yml
:concurrency: 5
:queues:
  - [critical, 3]  # Process 3x more often
  - [default, 2]
  - [low, 1]
  - [mailers, 2]

# In your jobs
class PaymentJob
  include Sidekiq::Job
  sidekiq_options queue: 'critical'
end

class ReportJob
  include Sidekiq::Job
  sidekiq_options queue: 'low'
end
```

## Job Design Patterns

### Jobs Should Defer to Service Layer
```ruby
# BAD - Business logic in job
class ChargeSubscriptionJob
  include Sidekiq::Job
  
  def perform(subscription_id)
    subscription = Subscription.find(subscription_id)
    
    charge = PaymentGateway.charge(
      amount: subscription.amount,
      customer: subscription.customer_token
    )
    
    if charge.success?
      subscription.update!(
        last_charged_at: Time.current,
        next_charge_at: 1.month.from_now
      )
      SubscriptionMailer.receipt(subscription, charge).deliver_later
    else
      subscription.update!(status: 'failed')
      AdminMailer.payment_failed(subscription).deliver_later
    end
  end
end

# GOOD - Job delegates to service
class ChargeSubscriptionJob
  include Sidekiq::Job
  
  sidekiq_options retry: 3
  
  def perform(subscription_id)
    subscription = Subscription.find(subscription_id)
    result = ChargeSubscription.new.call(subscription: subscription)
    
    if result.failure?
      # Let Sidekiq retry mechanism handle it
      raise result.error
    end
  rescue ActiveRecord::RecordNotFound
    # Subscription deleted, don't retry
  end
end
```

### Pass Simple Arguments
```ruby
# BAD - Passing complex objects
ProcessOrderJob.perform_async(order)  # Serializes entire object

# GOOD - Pass IDs
ProcessOrderJob.perform_async(order.id)

# GOOD - Pass simple data when record might be deleted
SendEmailJob.perform_async(
  'user_welcome',
  { email: user.email, name: user.name }
)
```

## Idempotency

### Make Jobs Idempotent
```ruby
class ProcessPaymentJob
  include Sidekiq::Job
  
  def perform(payment_id)
    payment = Payment.find(payment_id)
    
    # Check if already processed
    return if payment.processed?
    
    # Use database transaction with lock
    payment.with_lock do
      # Double-check after acquiring lock
      return if payment.processed?
      
      result = PaymentGateway.charge(payment)
      payment.update!(
        processed: true,
        gateway_id: result.transaction_id,
        processed_at: Time.current
      )
    end
  end
end
```

### Use Unique Jobs When Needed
```ruby
# Using sidekiq-unique-jobs gem
class SyncUserJob
  include Sidekiq::Job
  
  sidekiq_options(
    unique: :until_executed,
    unique_args: ->(args) { [args.first] }  # Unique per user_id
  )
  
  def perform(user_id)
    user = User.find(user_id)
    ExternalApi.sync_user(user)
  end
end
```

## Error Handling

### Let Jobs Fail and Retry
```ruby
# GOOD - Let Sidekiq handle retries
class FetchDataJob
  include Sidekiq::Job
  
  sidekiq_options retry: 5
  
  def perform(source_id)
    source = DataSource.find(source_id)
    data = ExternalApi.fetch(source.url)  # Might raise exception
    source.update!(last_fetched_data: data)
  end
end

# Configure exponential backoff
class ImportantJob
  include Sidekiq::Job
  
  sidekiq_options retry: 10
  
  sidekiq_retry_in do |count, exception|
    10 * (count ** 2)  # 10, 40, 90, 160 seconds...
  end
end
```

### Handle Specific Errors
```ruby
class SendNotificationJob
  include Sidekiq::Job
  
  def perform(user_id, message)
    user = User.find(user_id)
    NotificationService.send(user, message)
  rescue User::InvalidPhoneNumber => e
    # Don't retry for permanent failures
    logger.error "Invalid phone for user #{user_id}: #{e.message}"
  rescue Net::OpenTimeout => e
    # Let it retry for temporary failures
    raise
  end
end
```

## Job Testing

### Test Job Enqueueing
```ruby
RSpec.describe OrdersController do
  it "enqueues job after order creation" do
    expect {
      post :create, params: { order: attributes }
    }.to change { ProcessOrderJob.jobs.size }.by(1)
    
    job = ProcessOrderJob.jobs.last
    expect(job['args']).to eq([Order.last.id])
  end
end
```

### Test Job Execution
```ruby
RSpec.describe ProcessOrderJob do
  it "processes the order" do
    order = create(:order)
    
    expect(ProcessOrder).to receive(:new).and_call_original
    
    ProcessOrderJob.new.perform(order.id)
    
    expect(order.reload).to be_processed
  end
  
  it "handles missing orders gracefully" do
    expect {
      ProcessOrderJob.new.perform(999999)
    }.not_to raise_error
  end
end
```

### Test Idempotency
```ruby
RSpec.describe ChargePaymentJob do
  it "is idempotent" do
    payment = create(:payment)
    
    # First execution
    ChargePaymentJob.new.perform(payment.id)
    expect(payment.reload).to be_charged
    
    # Second execution should not double-charge
    expect(PaymentGateway).not_to receive(:charge)
    ChargePaymentJob.new.perform(payment.id)
  end
end
```

## Monitoring

### Add Instrumentation
```ruby
class CriticalJob
  include Sidekiq::Job
  
  def perform(id)
    start_time = Time.current
    
    # Job logic here
    process_record(id)
    
    duration = Time.current - start_time
    StatsD.timing('jobs.critical_job.duration', duration)
    StatsD.increment('jobs.critical_job.success')
  rescue => e
    StatsD.increment('jobs.critical_job.failure')
    raise
  end
end
```

### Dead Job Handler
```ruby
# config/initializers/sidekiq.rb
Sidekiq.configure_server do |config|
  config.death_handlers << ->(job, exception) do
    AdminMailer.job_failed(
      job['class'],
      job['args'],
      exception.message
    ).deliver_later
  end
end
```

## Best Practices Summary

1. **Use Sidekiq directly** instead of ActiveJob
2. **Pass simple arguments** (IDs, not objects)
3. **Make jobs idempotent** 
4. **Defer to service objects** for business logic
5. **Let jobs fail and retry** for transient errors
6. **Use appropriate queue priorities**
7. **Monitor job performance and failures**
8. **Test both enqueueing and execution**