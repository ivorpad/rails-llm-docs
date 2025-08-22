# Service Layer Patterns

## Core Principles
Services should be stateless, explicitly-named classes with explicitly-named methods that reveal behavior clearly.

## Naming Conventions

### Clear, Action-Oriented Names
```ruby
# GOOD - Clear what it does
class ChargeSubscription
class RegisterUser
class CalculateShipping

# BAD - Vague or generic
class UserService
class ProcessService
class Handler
```

### Method Names Match the Action
```ruby
# GOOD - Explicit method names
class ChargeSubscription
  def call(subscription:, amount:)
  end
end

# ACCEPTABLE - When action is clear from class name
class RegisterUser
  def register(email:, password:)
  end
end

# BAD - Generic or misleading
class UserService
  def process(data)
  end
end
```

## Rich Result Objects

### Return Result Objects, Not Booleans
```ruby
# BAD - Boolean return
class ChargeCard
  def call(card:, amount:)
    if charge_successful?
      true
    else
      false
    end
  end
end

# GOOD - Rich result object
class ChargeCard
  def call(card:, amount:)
    response = payment_gateway.charge(card, amount)
    
    if response.success?
      Result.success(
        transaction_id: response.id,
        amount: amount,
        charged_at: Time.current
      )
    else
      Result.failure(
        error_code: response.error_code,
        message: response.error_message
      )
    end
  end
end
```

### Simple Result Class Implementation
```ruby
class Result
  attr_reader :data, :error
  
  def self.success(data = {})
    new(success: true, data: data)
  end
  
  def self.failure(error)
    new(success: false, error: error)
  end
  
  def initialize(success:, data: nil, error: nil)
    @success = success
    @data = data
    @error = error
  end
  
  def success?
    @success
  end
  
  def failure?
    !@success
  end
end
```

## Service Parameters

### Receive Context and Data, Not Services
```ruby
# BAD - Receiving services as dependencies
class ProcessOrder
  def call(order:, payment_service:, email_service:)
    payment_service.charge(order.total)
    email_service.send_confirmation(order)
  end
end

# GOOD - Services handle their own dependencies
class ProcessOrder
  def call(order:, payment_method:)
    payment_result = charge_payment(order, payment_method)
    send_confirmation_email(order) if payment_result.success?
    
    payment_result
  end
  
  private
  
  def charge_payment(order, payment_method)
    PaymentGateway.charge(
      amount: order.total,
      payment_method: payment_method
    )
  end
  
  def send_confirmation_email(order)
    OrderMailer.confirmation(order).deliver_later
  end
end
```

## Stateless Services

### Avoid Instance Variables for State
```ruby
# BAD - Stateful service
class CalculateDiscount
  def initialize(order)
    @order = order
    @discount = 0
  end
  
  def calculate
    apply_quantity_discount
    apply_coupon_discount
    @discount
  end
  
  private
  
  def apply_quantity_discount
    @discount += @order.quantity * 0.1
  end
end

# GOOD - Stateless service
class CalculateDiscount
  def call(order:, coupon: nil)
    discount = 0
    discount += quantity_discount(order)
    discount += coupon_discount(coupon) if coupon
    
    Result.success(
      discount_amount: discount,
      final_price: order.total - discount
    )
  end
  
  private
  
  def quantity_discount(order)
    order.quantity * 0.1
  end
  
  def coupon_discount(coupon)
    coupon.discount_amount
  end
end
```

## Avoid Anti-patterns

### Don't Use Class Methods as Primary Interface
```ruby
# BAD - Class method limits flexibility
class ProcessPayment
  def self.call(order)
    # Can't easily test or extend
  end
end

# GOOD - Instance method allows flexibility
class ProcessPayment
  def call(order:)
    # Easier to test, mock, and extend
  end
end
```

### Don't Use Dependency Injection
```ruby
# BAD - Dependency injection obscures behavior
class CreateUser
  def initialize(validator:, repository:, mailer:)
    @validator = validator
    @repository = repository
    @mailer = mailer
  end
  
  def call(params)
    # Hard to understand what this actually does
  end
end

# GOOD - Direct dependencies are clear
class CreateUser
  def call(email:, password:)
    validation_result = validate_user_params(email, password)
    return validation_result if validation_result.failure?
    
    user = User.create!(email: email, password: password)
    UserMailer.welcome(user).deliver_later
    
    Result.success(user: user)
  end
end
```

## Usage Example
```ruby
# In controller
class UsersController < ApplicationController
  def create
    result = CreateUser.new.call(
      email: params[:email],
      password: params[:password]
    )
    
    if result.success?
      session[:user_id] = result.data[:user].id
      redirect_to dashboard_path
    else
      @error = result.error[:message]
      render :new
    end
  end
end

# In background job
class ProcessOrderJob < ApplicationJob
  def perform(order_id)
    order = Order.find(order_id)
    result = ProcessOrder.new.call(order: order)
    
    if result.failure?
      NotifyAdminService.new.call(
        error: result.error,
        context: "Order #{order_id} processing failed"
      )
    end
  end
end
```