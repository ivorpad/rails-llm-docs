# Validations vs Constraints

## Key Principle
Database constraints provide data integrity, validations provide user experience. Use both, but never rely on validations alone for data integrity.

## Why Validations Don't Provide Data Integrity

### 1. Validations Can Be Bypassed
```ruby
# All these bypass validations
user.save(validate: false)
user.update_attribute(:email, 'invalid')
user.update_column(:email, 'invalid')
User.update_all(email: 'invalid')

# Direct SQL bypasses everything
ActiveRecord::Base.connection.execute("UPDATE users SET email = NULL")
```

### 2. Race Conditions
```ruby
# Even with validation, race conditions can occur
class User < ApplicationRecord
  validates :email, uniqueness: true
end

# Two simultaneous requests can both pass validation
# and create duplicate emails without database constraint
```

### 3. External Systems
```ruby
# Other systems accessing your database won't run Rails validations
# - Database admin tools
# - ETL processes
# - Other applications sharing the database
# - Rails console with validate: false
```

## Database Constraints for Data Integrity

### Essential Constraints
```ruby
class CreateUsers < ActiveRecord::Migration[7.0]
  def change
    create_table :users do |t|
      # NOT NULL constraints
      t.string :email, null: false
      t.string :name, null: false
      
      # Unique constraints
      t.index :email, unique: true
      
      # Foreign key constraints
      t.references :company, null: false, foreign_key: true
      
      t.timestamps
    end
    
    # Check constraints
    execute <<-SQL
      ALTER TABLE users
      ADD CONSTRAINT valid_email
      CHECK (email ~* '^[A-Za-z0-9._%+-]+@[A-Za-z0-9.-]+\\.[A-Za-z]{2,}$')
    SQL
  end
end
```

### Complex Business Rules
```ruby
# Enforce one active subscription per user at database level
add_index :subscriptions, 
          [:user_id, :status], 
          unique: true,
          where: "status = 'active'",
          name: 'idx_one_active_subscription'

# Ensure price is positive
execute <<-SQL
  ALTER TABLE products
  ADD CONSTRAINT positive_price
  CHECK (price > 0)
SQL

# Ensure date ranges are valid
execute <<-SQL
  ALTER TABLE events
  ADD CONSTRAINT valid_date_range
  CHECK (starts_at < ends_at)
SQL
```

## Validations for User Experience

### Provide Friendly Error Messages
```ruby
class User < ApplicationRecord
  # These provide good UX, but don't ensure data integrity
  validates :email, 
            presence: { message: "can't be blank" },
            format: { 
              with: URI::MailTo::EMAIL_REGEXP,
              message: "must be a valid email address"
            }
  
  validates :age, 
            numericality: { 
              greater_than_or_equal_to: 18,
              message: "must be 18 or older"
            }
  
  validates :password,
            length: { 
              minimum: 8,
              message: "must be at least 8 characters"
            },
            format: {
              with: /\A(?=.*[a-z])(?=.*[A-Z])(?=.*\d)/,
              message: "must include uppercase, lowercase, and number"
            }
end
```

### Custom Validations for Complex Rules
```ruby
class Subscription < ApplicationRecord
  validate :user_can_subscribe
  validate :plan_available_in_region
  
  private
  
  def user_can_subscribe
    if user.subscriptions.active.exists?
      errors.add(:base, "You already have an active subscription")
    end
    
    if plan.requires_verification && !user.verified?
      errors.add(:base, "Account verification required for this plan")
    end
  end
  
  def plan_available_in_region
    unless plan.available_regions.include?(user.region)
      errors.add(:plan, "is not available in your region")
    end
  end
end
```

## Combining Both Approaches

### Example: User Registration
```ruby
# Migration with constraints
class CreateUsers < ActiveRecord::Migration[7.0]
  def change
    create_table :users do |t|
      t.string :email, null: false
      t.string :username, null: false
      t.string :name, null: false
      t.integer :age, null: false
      
      t.timestamps
    end
    
    add_index :users, :email, unique: true
    add_index :users, :username, unique: true
    
    execute <<-SQL
      ALTER TABLE users
      ADD CONSTRAINT valid_age CHECK (age >= 0 AND age <= 150)
    SQL
  end
end

# Model with validations for UX
class User < ApplicationRecord
  # Presence validations (redundant with NOT NULL, but better errors)
  validates :email, presence: true
  validates :username, presence: true
  validates :name, presence: true
  
  # Uniqueness validations (redundant with index, but better errors)
  validates :email, uniqueness: { case_sensitive: false }
  validates :username, uniqueness: { case_sensitive: false }
  
  # Format validations for better UX
  validates :email, format: { with: URI::MailTo::EMAIL_REGEXP }
  validates :username, format: { 
    with: /\A[a-zA-Z0-9_]+\z/,
    message: "can only contain letters, numbers, and underscores"
  }
  
  # Business rules
  validates :age, numericality: { 
    greater_than_or_equal_to: 13,
    message: "must be 13 or older to register"
  }
end
```

## Testing Strategy

### Test Constraints Separately
```ruby
RSpec.describe User do
  describe "database constraints" do
    it "enforces email uniqueness at database level" do
      User.create!(email: "test@example.com", name: "Test")
      duplicate = User.new(email: "test@example.com", name: "Other")
      
      expect { duplicate.save!(validate: false) }
        .to raise_error(ActiveRecord::RecordNotUnique)
    end
    
    it "enforces NOT NULL on email" do
      user = User.new(name: "Test")
      
      expect { user.save!(validate: false) }
        .to raise_error(ActiveRecord::NotNullViolation)
    end
  end
  
  describe "validations" do
    it "provides friendly error for duplicate email" do
      User.create!(email: "test@example.com", name: "Test")
      duplicate = User.new(email: "test@example.com", name: "Other")
      
      expect(duplicate).not_to be_valid
      expect(duplicate.errors[:email]).to include("has already been taken")
    end
  end
end
```

## Best Practices

1. **Always use database constraints for critical data integrity**
2. **Add validations for better error messages**
3. **Don't trust validations alone for uniqueness**
4. **Test both constraints and validations**
5. **Document why both exist when it might seem redundant**

```ruby
class Product < ApplicationRecord
  # Database ensures SKU is unique (see migration)
  # This validation provides a better error message
  validates :sku, uniqueness: {
    message: "already exists. Each product must have a unique SKU."
  }
  
  # Database ensures price > 0 (see migration)  
  # This validation provides helpful context
  validates :price, numericality: {
    greater_than: 0,
    message: "must be greater than 0. Free products should use a different status."
  }
end
```