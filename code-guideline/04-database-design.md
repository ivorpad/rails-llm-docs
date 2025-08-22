# Database Design Best Practices

## Core Principle
The database should enforce correctness through constraints, not rely solely on application code. Design for data integrity first, performance second.

## Use SQL Schema Format
```ruby
# config/application.rb
config.active_record.schema_format = :sql
```
This preserves database-specific features like constraints, triggers, and functions.

## Column Type Selection

### Use Appropriate Types
```ruby
# BAD - Using strings for everything
create_table :products do |t|
  t.string :price        # Should be decimal
  t.string :quantity     # Should be integer
  t.string :released_on  # Should be date
end

# GOOD - Proper column types
create_table :products do |t|
  t.decimal :price, precision: 10, scale: 2
  t.integer :quantity, null: false, default: 0
  t.date :released_on
  t.boolean :active, null: false, default: true
end
```

### Always Use TIMESTAMP WITH TIME ZONE
```ruby
# BAD - Timezone-naive timestamps
create_table :events do |t|
  t.datetime :starts_at
end

# GOOD - Timezone-aware timestamps
create_table :events do |t|
  t.column :starts_at, :timestamptz, null: false
  t.column :ends_at, :timestamptz, null: false
end

# Or in raw SQL migration
execute <<-SQL
  ALTER TABLE events 
  ADD COLUMN starts_at TIMESTAMP WITH TIME ZONE NOT NULL
SQL
```

## Database Constraints

### NOT NULL Constraints
```ruby
# Enforce required fields at database level
create_table :users do |t|
  t.string :email, null: false
  t.string :name, null: false
  t.boolean :active, null: false, default: true
end
```

### Unique Constraints
```ruby
# Single column uniqueness
add_index :users, :email, unique: true

# Composite uniqueness
add_index :subscriptions, [:user_id, :plan_id], 
          unique: true, 
          where: "status = 'active'",
          name: 'idx_one_active_subscription_per_user'
```

### Foreign Key Constraints
```ruby
# Always add foreign keys
create_table :orders do |t|
  t.references :user, null: false, foreign_key: true
  t.references :product, null: false, foreign_key: { on_delete: :restrict }
end

# With custom behavior
add_foreign_key :comments, :users, on_delete: :cascade
add_foreign_key :orders, :products, on_delete: :restrict
```

### Check Constraints
```ruby
# Ensure data validity
class AddPriceConstraintToProducts < ActiveRecord::Migration[7.0]
  def up
    execute <<-SQL
      ALTER TABLE products 
      ADD CONSTRAINT positive_price 
      CHECK (price > 0)
    SQL
  end
  
  def down
    execute <<-SQL
      ALTER TABLE products 
      DROP CONSTRAINT positive_price
    SQL
  end
end

# Multiple conditions
execute <<-SQL
  ALTER TABLE events
  ADD CONSTRAINT valid_date_range
  CHECK (starts_at < ends_at)
SQL

# Enum-like constraints
execute <<-SQL
  ALTER TABLE orders
  ADD CONSTRAINT valid_status
  CHECK (status IN ('pending', 'processing', 'completed', 'cancelled'))
SQL
```

## Lookup Tables vs Enums

### Use Lookup Tables for Business Entities
```ruby
# GOOD - Lookup table for data that might change
create_table :order_statuses do |t|
  t.string :name, null: false
  t.string :display_name, null: false
  t.text :description
  t.integer :sort_order, null: false, default: 0
  t.timestamps
end

add_index :order_statuses, :name, unique: true

create_table :orders do |t|
  t.references :order_status, null: false, foreign_key: true
end

# Seed the lookup data
OrderStatus.create!([
  { name: 'pending', display_name: 'Pending', sort_order: 1 },
  { name: 'processing', display_name: 'Processing', sort_order: 2 },
  { name: 'shipped', display_name: 'Shipped', sort_order: 3 }
])
```

### Use Check Constraints for Simple Enums
```ruby
# GOOD - Check constraint for simple, unchanging values
create_table :users do |t|
  t.string :role, null: false, default: 'member'
end

execute <<-SQL
  ALTER TABLE users
  ADD CONSTRAINT valid_role
  CHECK (role IN ('admin', 'moderator', 'member'))
SQL
```

## Index Strategy

### Index Foreign Keys
```ruby
# Always index foreign keys
create_table :comments do |t|
  t.references :user, null: false, foreign_key: true, index: true
  t.references :post, null: false, foreign_key: true, index: true
end
```

### Composite Indexes for Common Queries
```ruby
# Index columns used together in WHERE clauses
add_index :orders, [:user_id, :created_at]
add_index :products, [:category_id, :active, :created_at]
```

### Partial Indexes for Filtered Queries
```ruby
# Index only relevant rows
add_index :users, :email, where: "deleted_at IS NULL"
add_index :orders, :created_at, where: "status = 'pending'"
```

## Default Values

### Database Defaults Over Application Defaults
```ruby
# GOOD - Database ensures defaults
create_table :products do |t|
  t.integer :quantity, null: false, default: 0
  t.boolean :active, null: false, default: true
  t.decimal :tax_rate, null: false, default: 0.0
end

# BAD - Relying on Rails callbacks
class Product < ApplicationRecord
  before_validation :set_defaults
  
  def set_defaults
    self.quantity ||= 0
    self.active = true if active.nil?
  end
end
```

## Migration Best Practices

### Reversible Migrations
```ruby
class AddConstraintToOrders < ActiveRecord::Migration[7.0]
  def up
    execute <<-SQL
      ALTER TABLE orders
      ADD CONSTRAINT positive_total
      CHECK (total >= 0)
    SQL
  end
  
  def down
    execute <<-SQL
      ALTER TABLE orders
      DROP CONSTRAINT positive_total
    SQL
  end
end
```

### Safe Column Additions
```ruby
# Add column with default value safely
class AddActiveToUsers < ActiveRecord::Migration[7.0]
  def change
    # First add nullable column
    add_column :users, :active, :boolean
    
    # Backfill existing records
    User.update_all(active: true)
    
    # Then add constraints
    change_column_null :users, :active, false
    change_column_default :users, :active, true
  end
end
```

## Testing Database Constraints
```ruby
# Test that constraints work
RSpec.describe Product do
  it "enforces positive price constraint" do
    product = Product.new(price: -10)
    
    expect { product.save!(validate: false) }
      .to raise_error(ActiveRecord::StatementInvalid, /positive_price/)
  end
  
  it "enforces foreign key constraint" do
    order = Order.new(user_id: 999999)
    
    expect { order.save!(validate: false) }
      .to raise_error(ActiveRecord::InvalidForeignKey)
  end
end
```