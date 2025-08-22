# Active Record Patterns

## Core Principle
Active Record should be used for database access only. Use class methods for reusable queries and instance methods for simple derived attributes from database columns.

## Class Methods for Database Operations

### Scopes for Common Queries
```ruby
class User < ApplicationRecord
  # Simple scopes
  scope :active, -> { where(active: true) }
  scope :recent, -> { order(created_at: :desc) }
  
  # Parameterized scopes
  scope :created_after, ->(date) { where('created_at > ?', date) }
  scope :with_role, ->(role) { where(role: role) }
  
  # Combining scopes
  scope :active_admins, -> { active.with_role('admin') }
end

# Usage
User.active.recent.limit(10)
User.created_after(30.days.ago)
```

### Class Methods for Complex Queries
```ruby
class Order < ApplicationRecord
  # Use class methods when logic is complex
  def self.pending_for_user(user)
    where(user: user, status: 'pending')
      .includes(:line_items)
      .order(created_at: :desc)
  end
  
  def self.with_total_greater_than(amount)
    joins(:line_items)
      .group('orders.id')
      .having('SUM(line_items.amount) > ?', amount)
  end
  
  def self.by_month(date)
    start_date = date.beginning_of_month
    end_date = date.end_of_month
    where(created_at: start_date..end_date)
  end
end
```

### Query Objects for Very Complex Queries
```ruby
# app/queries/orders_query.rb
class OrdersQuery
  def initialize(relation = Order.all)
    @relation = relation
  end
  
  def with_filters(params)
    result = @relation
    result = filter_by_status(result, params[:status])
    result = filter_by_date_range(result, params[:from], params[:to])
    result = filter_by_customer(result, params[:customer_id])
    result
  end
  
  private
  
  def filter_by_status(relation, status)
    return relation unless status.present?
    relation.where(status: status)
  end
  
  def filter_by_date_range(relation, from, to)
    relation = relation.where('created_at >= ?', from) if from.present?
    relation = relation.where('created_at <= ?', to) if to.present?
    relation
  end
  
  def filter_by_customer(relation, customer_id)
    return relation unless customer_id.present?
    relation.where(customer_id: customer_id)
  end
end

# Usage
OrdersQuery.new.with_filters(params)
```

## Instance Methods for Derived Attributes

### Simple Calculated Fields
```ruby
class Product < ApplicationRecord
  # Good - Simple derivations from database columns
  def display_name
    "#{brand} - #{name}"
  end
  
  def discounted?
    discount_percentage > 0
  end
  
  def price_with_tax
    price * (1 + tax_rate)
  end
  
  # Good - Database-backed calculations
  def in_stock?
    quantity > 0
  end
  
  def low_stock?
    quantity.between?(1, 10)
  end
end
```

### Avoid Complex Business Logic
```ruby
class Order < ApplicationRecord
  # GOOD - Simple database-derived method
  def total
    line_items.sum(:amount)
  end
  
  def item_count
    line_items.count
  end
  
  # BAD - Business logic doesn't belong here
  def calculate_shipping
    # Complex shipping calculation
  end
  
  def apply_discount(coupon)
    # Discount application logic
  end
end

# Business logic goes in services
class CalculateShipping
  def call(order:)
    # Shipping logic here
  end
end
```

## Association Best Practices

### Use `has_many :through` for Many-to-Many
```ruby
# GOOD - Explicit join model
class User < ApplicationRecord
  has_many :memberships
  has_many :organizations, through: :memberships
end

class Membership < ApplicationRecord
  belongs_to :user
  belongs_to :organization
  
  # Can add attributes to the relationship
  validates :role, presence: true
end

class Organization < ApplicationRecord
  has_many :memberships
  has_many :users, through: :memberships
end
```

### Leverage Association Extensions
```ruby
class Order < ApplicationRecord
  has_many :line_items do
    def total_amount
      sum(:amount)
    end
    
    def by_product(product)
      where(product: product)
    end
  end
end

# Usage
order.line_items.total_amount
order.line_items.by_product(product)
```

### Use Counter Caches for Performance
```ruby
class Comment < ApplicationRecord
  belongs_to :post, counter_cache: true
end

class Post < ApplicationRecord
  has_many :comments
  # Requires comments_count column
end

# Migration
add_column :posts, :comments_count, :integer, default: 0, null: false

# Reset counters if adding to existing data
Post.reset_counters(post.id, :comments)
```

## Efficient Loading Strategies

### Use `includes` to Avoid N+1 Queries
```ruby
# BAD - N+1 query problem
users = User.all
users.each do |user|
  puts user.posts.count  # Queries database each iteration
end

# GOOD - Eager loading
users = User.includes(:posts)
users.each do |user|
  puts user.posts.size  # No additional queries
end
```

### Use `joins` for Filtering
```ruby
# When you need to filter but don't need the associated data
User.joins(:posts).where(posts: { published: true }).distinct

# vs includes which loads all data
User.includes(:posts).where(posts: { published: true })
```

### Use `preload` and `eager_load` When Appropriate
```ruby
# preload - Separate queries
User.preload(:posts)
# SELECT * FROM users
# SELECT * FROM posts WHERE user_id IN (...)

# eager_load - Single LEFT JOIN query
User.eager_load(:posts)
# SELECT * FROM users LEFT JOIN posts ON ...

# includes - Rails decides based on conditions
User.includes(:posts).where(posts: { published: true })
# Will use eager_load due to WHERE condition
```

## Batch Processing
```ruby
# Process records in batches to avoid memory issues
User.find_each(batch_size: 100) do |user|
  ProcessUserJob.perform_later(user)
end

# With conditions
User.active.find_in_batches(batch_size: 500) do |users|
  users.each do |user|
    # Process user
  end
end
```

## Model Organization
```ruby
class User < ApplicationRecord
  # Constants
  ROLES = %w[admin moderator member].freeze
  
  # Concerns
  include Searchable
  include Timestampable
  
  # Associations
  belongs_to :company
  has_many :posts
  has_many :comments
  
  # Validations
  validates :email, presence: true, uniqueness: true
  validates :role, inclusion: { in: ROLES }
  
  # Scopes
  scope :active, -> { where(active: true) }
  scope :by_role, ->(role) { where(role: role) }
  
  # Class methods
  def self.search(query)
    where('name LIKE ? OR email LIKE ?', "%#{query}%", "%#{query}%")
  end
  
  # Instance methods
  def display_name
    name.presence || email.split('@').first
  end
end
```