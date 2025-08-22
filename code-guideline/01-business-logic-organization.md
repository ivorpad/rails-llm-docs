# Business Logic Organization

## Key Principle
Business logic should live in a dedicated service layer, not in Active Records or Controllers. This creates a clear separation of concerns and makes code more maintainable.

## Why This Matters
- Business logic is complex and changes frequently (high churn)
- Bugs in commonly-used classes have wide effects (high fan-in)
- Active Records with business logic become critical failure points

## Best Practices

### 1. Keep Business Logic Out of Models
```ruby
# BAD - Business logic in Active Record
class Order < ApplicationRecord
  def complete_purchase
    transaction do
      update!(status: 'completed')
      charge_credit_card
      send_confirmation_email
      update_inventory
    end
  end
end

# GOOD - Business logic in service
class CompletePurchase
  def call(order:, payment_method:)
    Order.transaction do
      order.update!(status: 'completed')
      payment_result = PaymentProcessor.charge(payment_method, order.total)
      EmailService.send_confirmation(order)
      InventoryService.update_for_order(order)
      
      Result.new(success: true, order: order, payment: payment_result)
    end
  end
end
```

### 2. Create a Service Layer
Organize business logic into service objects that:
- Have single responsibilities
- Are stateless
- Return rich result objects
- Have explicit, descriptive names

```ruby
# app/services/orders/complete_purchase.rb
module Orders
  class CompletePurchase
    def call(order:, payment_method:)
      # Business logic here
    end
  end
end
```

### 3. Service Organization Structure
```
app/
  services/
    orders/
      complete_purchase.rb
      calculate_shipping.rb
      apply_discount.rb
    users/
      authenticate.rb
      register.rb
      reset_password.rb
    payments/
      process_payment.rb
      refund_payment.rb
```

### 4. Keep Controllers Thin
Controllers should only:
- Handle HTTP concerns
- Call service objects
- Render responses

```ruby
class OrdersController < ApplicationController
  def complete
    result = Orders::CompletePurchase.call(
      order: @order,
      payment_method: payment_params
    )
    
    if result.success?
      redirect_to result.order
    else
      render :show, alert: result.error_message
    end
  end
end
```

### 5. Active Records for Database Access Only
Models should only contain:
- Associations
- Database-level validations
- Scopes for common queries
- Simple derived attributes

```ruby
class Order < ApplicationRecord
  belongs_to :user
  has_many :line_items
  
  scope :completed, -> { where(status: 'completed') }
  
  def total
    line_items.sum(:amount)
  end
end
```

## Benefits
- **Testability**: Service objects are easier to test in isolation
- **Reusability**: Business logic can be called from controllers, jobs, rake tasks
- **Clarity**: Each service has a clear, single purpose
- **Maintainability**: Changes to business logic don't affect data models