# Avoiding Fat Models

## The Problem with Fat Models
"Fat models, skinny controllers" was well-intentioned advice that led to bloated Active Record classes containing business logic, callbacks, validations, and complex methods. This creates high-risk, hard-to-test code.

## Why Fat Models Are Problematic
- **High Fan-in**: Models are used everywhere, bugs affect entire app
- **Mixed Concerns**: Database access mixed with business logic
- **Testing Difficulty**: Database required for all tests
- **Callback Hell**: Complex callback chains become unmaintainable

## What Belongs in Models

### YES: Database-Related Code
```ruby
class User < ApplicationRecord
  # Associations
  has_many :orders
  belongs_to :company
  
  # Database-level validations
  validates :email, presence: true, uniqueness: true
  
  # Scopes for common queries
  scope :active, -> { where(active: true) }
  scope :by_signup_date, -> { order(created_at: :desc) }
  
  # Simple derived attributes from database columns
  def full_name
    "#{first_name} #{last_name}"
  end
end
```

### NO: Business Logic
```ruby
# BAD - Business logic in model
class Order < ApplicationRecord
  after_create :send_confirmation_email
  after_save :update_inventory
  before_destroy :check_if_can_cancel
  
  def process_payment
    # Complex payment logic
  end
  
  def calculate_tax
    # Tax calculation logic
  end
  
  def apply_discounts
    # Discount logic
  end
end

# GOOD - Keep model simple
class Order < ApplicationRecord
  belongs_to :user
  has_many :line_items
  
  validates :number, presence: true, uniqueness: true
  
  scope :pending, -> { where(status: 'pending') }
  
  def total
    line_items.sum(:amount)
  end
end

# Move business logic to services
class ProcessOrderPayment
  def call(order:, payment_method:)
    # Payment logic here
  end
end
```

## Refactoring Fat Models

### Step 1: Extract Callbacks to Services
```ruby
# BEFORE - Callbacks in model
class User < ApplicationRecord
  after_create :send_welcome_email
  after_update :sync_with_crm
  
  private
  
  def send_welcome_email
    UserMailer.welcome(self).deliver_later
  end
  
  def sync_with_crm
    CrmSyncJob.perform_later(self)
  end
end

# AFTER - Logic in services
class User < ApplicationRecord
  # No callbacks
end

class CreateUser
  def call(email:, password:)
    user = User.create!(email: email, password: password)
    send_welcome_email(user)
    sync_with_crm(user)
    
    Result.success(user: user)
  end
  
  private
  
  def send_welcome_email(user)
    UserMailer.welcome(user).deliver_later
  end
  
  def sync_with_crm(user)
    CrmSyncJob.perform_later(user)
  end
end
```

### Step 2: Extract Complex Methods
```ruby
# BEFORE - Complex method in model
class Product < ApplicationRecord
  def available_for_purchase?
    in_stock? && 
      !discontinued? && 
      price.present? && 
      published? &&
      !restricted_in_region?(current_region)
  end
  
  def calculate_final_price(user)
    base_price = price
    base_price -= loyalty_discount(user)
    base_price -= seasonal_discount
    base_price += calculate_tax(user.region)
    base_price
  end
end

# AFTER - Simple model, logic in service
class Product < ApplicationRecord
  scope :in_stock, -> { where('quantity > 0') }
  scope :published, -> { where(published: true) }
  
  def in_stock?
    quantity > 0
  end
end

class ProductAvailability
  def available_for_purchase?(product:, region:)
    product.in_stock? &&
      !product.discontinued? &&
      product.price.present? &&
      product.published? &&
      !restricted_in_region?(product, region)
  end
  
  private
  
  def restricted_in_region?(product, region)
    # Check region restrictions
  end
end

class CalculateProductPrice
  def call(product:, user:)
    base_price = product.price
    base_price -= loyalty_discount(user, product)
    base_price -= seasonal_discount(product)
    base_price += calculate_tax(base_price, user.region)
    
    Result.success(
      final_price: base_price,
      discount_applied: product.price - base_price
    )
  end
end
```

### Step 3: Extract Validations That Are Business Rules
```ruby
# BEFORE - Business rule validations in model
class Subscription < ApplicationRecord
  validate :can_only_have_one_active_subscription
  validate :trial_only_once_per_user
  validate :plan_available_in_region
  
  private
  
  def can_only_have_one_active_subscription
    # Complex validation logic
  end
end

# AFTER - Keep simple validations, extract business rules
class Subscription < ApplicationRecord
  validates :plan_id, presence: true
  validates :user_id, presence: true
end

class CreateSubscription
  def call(user:, plan:)
    validation_result = validate_subscription(user, plan)
    return validation_result if validation_result.failure?
    
    subscription = Subscription.create!(
      user: user,
      plan: plan,
      status: 'active'
    )
    
    Result.success(subscription: subscription)
  end
  
  private
  
  def validate_subscription(user, plan)
    return Result.failure(error: 'Already has active subscription') if user.subscriptions.active.any?
    return Result.failure(error: 'Trial already used') if user.had_trial? && plan.trial?
    return Result.failure(error: 'Plan not available in region') unless plan.available_in?(user.region)
    
    Result.success
  end
end
```

## Benefits of Thin Models
- **Single Responsibility**: Models handle database, services handle business logic
- **Testability**: Services can be tested without database
- **Flexibility**: Business logic can be composed and reused
- **Clarity**: Easy to understand what each class does
- **Maintainability**: Changes don't cascade through the entire app