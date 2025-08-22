# Migration Best Practices

## Core Principles
Migrations should be safe, reversible, and maintain data integrity at all times. Think about existing data and production deployments.

## Safe Migration Patterns

### Adding Columns Safely
```ruby
# BAD - Can cause issues with existing records
class AddActiveToUsers < ActiveRecord::Migration[7.0]
  def change
    add_column :users, :active, :boolean, null: false, default: true
  end
end

# GOOD - Multi-step approach
class AddActiveToUsers < ActiveRecord::Migration[7.0]
  def up
    # Step 1: Add nullable column
    add_column :users, :active, :boolean
    
    # Step 2: Backfill existing records
    User.reset_column_information
    User.update_all(active: true)
    
    # Step 3: Add NOT NULL constraint
    change_column_null :users, :active, false
    change_column_default :users, :active, true
  end
  
  def down
    remove_column :users, :active
  end
end
```

### Renaming Columns Safely
```ruby
# BAD - Breaks running application
class RenameEmailToEmailAddress < ActiveRecord::Migration[7.0]
  def change
    rename_column :users, :email, :email_address
  end
end

# GOOD - Multi-deployment approach
# Deployment 1: Add new column and sync data
class AddEmailAddressToUsers < ActiveRecord::Migration[7.0]
  def up
    add_column :users, :email_address, :string
    
    # Copy existing data
    User.reset_column_information
    User.find_each do |user|
      user.update_column(:email_address, user.email)
    end
    
    # Add constraints to new column
    change_column_null :users, :email_address, false
    add_index :users, :email_address, unique: true
  end
  
  def down
    remove_column :users, :email_address
  end
end

# In model during transition
class User < ApplicationRecord
  def email=(value)
    super
    self.email_address = value
  end
end

# Deployment 2: Remove old column
class RemoveEmailFromUsers < ActiveRecord::Migration[7.0]
  def change
    remove_column :users, :email, :string
  end
end
```

## Adding Constraints

### Adding NOT NULL Constraints
```ruby
class AddNotNullToUserEmail < ActiveRecord::Migration[7.0]
  def up
    # First ensure no NULL values exist
    User.where(email: nil).update_all(email: 'placeholder@example.com')
    
    # Then add constraint
    change_column_null :users, :email, false
  end
  
  def down
    change_column_null :users, :email, true
  end
end
```

### Adding Foreign Keys
```ruby
class AddForeignKeyToOrders < ActiveRecord::Migration[7.0]
  def up
    # Clean up orphaned records first
    Order.where.not(user_id: User.select(:id)).destroy_all
    
    # Add foreign key with appropriate action
    add_foreign_key :orders, :users, on_delete: :restrict
  end
  
  def down
    remove_foreign_key :orders, :users
  end
end
```

### Adding Check Constraints
```ruby
class AddPriceConstraintToProducts < ActiveRecord::Migration[7.0]
  def up
    # Fix invalid data first
    Product.where('price <= ?', 0).update_all(price: 0.01)
    
    # Add constraint with validation
    execute <<-SQL
      ALTER TABLE products
      ADD CONSTRAINT positive_price
      CHECK (price > 0)
      NOT VALID
    SQL
    
    # Validate constraint separately (can be done online in Postgres 9.4+)
    execute <<-SQL
      ALTER TABLE products
      VALIDATE CONSTRAINT positive_price
    SQL
  end
  
  def down
    execute <<-SQL
      ALTER TABLE products
      DROP CONSTRAINT positive_price
    SQL
  end
end
```

## Index Management

### Adding Indexes Without Locking
```ruby
class AddIndexToUsersEmail < ActiveRecord::Migration[7.0]
  disable_ddl_transaction!
  
  def up
    add_index :users, :email, unique: true, algorithm: :concurrently
  end
  
  def down
    remove_index :users, :email, algorithm: :concurrently
  end
end
```

### Composite and Partial Indexes
```ruby
class AddCompositeIndexToOrders < ActiveRecord::Migration[7.0]
  def up
    # Composite index for common queries
    add_index :orders, [:user_id, :status, :created_at], 
              name: 'idx_orders_user_status_date'
    
    # Partial index for specific conditions
    add_index :orders, :created_at,
              where: "status = 'pending'",
              name: 'idx_pending_orders_by_date'
  end
  
  def down
    remove_index :orders, name: 'idx_orders_user_status_date'
    remove_index :orders, name: 'idx_pending_orders_by_date'
  end
end
```

## Data Migrations

### Separate Schema and Data Migrations
```ruby
# BAD - Mixing schema and data changes
class AddRoleToUsers < ActiveRecord::Migration[7.0]
  def change
    add_column :users, :role, :string
    User.update_all(role: 'member')
  end
end

# GOOD - Separate migrations
# 1. Schema migration
class AddRoleToUsers < ActiveRecord::Migration[7.0]
  def change
    add_column :users, :role, :string
  end
end

# 2. Data migration (separate file)
class BackfillUserRoles < ActiveRecord::Migration[7.0]
  def up
    User.reset_column_information
    User.find_each(batch_size: 1000) do |user|
      user.update_column(:role, determine_role(user))
    end
  end
  
  def down
    # Data migrations often aren't reversible
    raise ActiveRecord::IrreversibleMigration
  end
  
  private
  
  def determine_role(user)
    return 'admin' if user.email.ends_with?('@company.com')
    'member'
  end
end
```

## Reversible Migrations

### Always Make Migrations Reversible
```ruby
class CreateOrderStatuses < ActiveRecord::Migration[7.0]
  def up
    create_table :order_statuses do |t|
      t.string :name, null: false
      t.string :color, null: false
      t.integer :position, null: false
      t.timestamps
    end
    
    add_index :order_statuses, :name, unique: true
    
    # Seed initial data
    execute <<-SQL
      INSERT INTO order_statuses (name, color, position, created_at, updated_at)
      VALUES 
        ('pending', 'yellow', 1, NOW(), NOW()),
        ('processing', 'blue', 2, NOW(), NOW()),
        ('completed', 'green', 3, NOW(), NOW())
    SQL
  end
  
  def down
    drop_table :order_statuses
  end
end
```

### Using `reversible` Block
```ruby
class AddConstraintToProducts < ActiveRecord::Migration[7.0]
  def change
    reversible do |direction|
      direction.up do
        execute <<-SQL
          ALTER TABLE products
          ADD CONSTRAINT valid_sku
          CHECK (sku ~ '^[A-Z0-9]{8,}$')
        SQL
      end
      
      direction.down do
        execute <<-SQL
          ALTER TABLE products
          DROP CONSTRAINT valid_sku
        SQL
      end
    end
  end
end
```

## Migration Scripts

### Create Helper Scripts
```ruby
# bin/migrate-safe
#!/usr/bin/env ruby
require_relative '../config/environment'

# Check for pending migrations
if ActiveRecord::Base.connection.migration_context.needs_migration?
  puts "Running migrations..."
  
  # Take backup
  system("pg_dump -h localhost -U postgres myapp_production > backup_#{Time.now.to_i}.sql")
  
  # Run migrations
  ActiveRecord::Tasks::DatabaseTasks.migrate
  
  # Run data integrity checks
  puts "Checking data integrity..."
  User.find_each { |u| raise "Invalid user: #{u.id}" unless u.valid? }
  
  puts "Migration completed successfully!"
else
  puts "No pending migrations"
end
```

## Best Practices Summary

1. **Always use `up` and `down` methods for complex migrations**
2. **Test rollback in development before deploying**
3. **Use `disable_ddl_transaction!` for concurrent index creation**
4. **Clean up bad data before adding constraints**
5. **Separate schema changes from data changes**
6. **Consider multi-step deployments for breaking changes**
7. **Use SQL schema format to preserve all database features**

```ruby
# config/application.rb
config.active_record.schema_format = :sql
```