# Testing Strategy

## Core Principles
Understand the value and cost of tests. Test behavior, not implementation. Focus on tests that give confidence with reasonable maintenance cost.

## Testing Pyramid

### System Tests (Few)
- End-to-end user flows
- Critical business paths
- Integration between components

### Service/Integration Tests (Some)
- Business logic in services
- API endpoints
- Complex database queries

### Unit Tests (Many)
- Models with complex validations
- Service objects
- Helper methods
- Value objects

## What to Test and What Not to Test

### DO Test
```ruby
# Business logic
class CalculatePricing
  def call(order:)
    # Complex pricing logic worth testing
  end
end

# Complex validations
class Subscription < ApplicationRecord
  validate :cannot_exceed_plan_limits
  # Test this custom validation
end

# API responses
class Api::V1::ProductsController
  # Test response format and status codes
end

# Critical user flows
# Test: User can complete purchase
# Test: Admin can refund order
```

### DON'T Test
```ruby
# Simple delegations
class User < ApplicationRecord
  def full_name
    "#{first_name} #{last_name}"  # Too simple
  end
end

# Framework features
class Product < ApplicationRecord
  validates :name, presence: true  # Rails already tests this
  belongs_to :category  # Don't test associations
end

# Simple controllers without logic
class ProductsController < ApplicationController
  def index
    @products = Product.all  # Nothing to test
  end
end
```

## System Tests Strategy

### Use Rack Test for Non-JavaScript
```ruby
# spec/system/products_spec.rb
RSpec.describe "Products", type: :system do
  before do
    driven_by :rack_test  # Fast, no browser needed
  end
  
  it "allows browsing products" do
    product = create(:product, name: "Widget")
    
    visit products_path
    
    expect(page).to have_content("Widget")
    
    click_link "Widget"
    
    expect(page).to have_current_path(product_path(product))
    expect(page).to have_content(product.description)
  end
end
```

### Use Headless Chrome for JavaScript
```ruby
RSpec.describe "Interactive Features", type: :system do
  before do
    driven_by :selenium_chrome_headless
  end
  
  it "filters products dynamically" do
    create(:product, name: "Widget", category: "Hardware")
    create(:product, name: "Gadget", category: "Software")
    
    visit products_path
    
    select "Hardware", from: "category_filter"
    
    expect(page).to have_content("Widget")
    expect(page).not_to have_content("Gadget")
  end
end
```

## Service Object Testing

### Test Business Logic Thoroughly
```ruby
RSpec.describe CreateOrder do
  let(:service) { described_class.new }
  
  describe "#call" do
    context "with valid params" do
      it "creates an order" do
        user = create(:user)
        product = create(:product, price: 100)
        
        result = service.call(
          user: user,
          items: [{ product_id: product.id, quantity: 2 }]
        )
        
        expect(result).to be_success
        expect(result.order.total).to eq(200)
        expect(result.order.line_items.count).to eq(1)
      end
      
      it "applies discounts" do
        user = create(:user, :premium)
        product = create(:product, price: 100)
        
        result = service.call(
          user: user,
          items: [{ product_id: product.id, quantity: 1 }],
          coupon_code: "SAVE10"
        )
        
        expect(result.order.discount_amount).to eq(10)
        expect(result.order.total).to eq(90)
      end
    end
    
    context "with invalid params" do
      it "fails when product is out of stock" do
        product = create(:product, quantity: 0)
        
        result = service.call(
          user: create(:user),
          items: [{ product_id: product.id, quantity: 1 }]
        )
        
        expect(result).to be_failure
        expect(result.error).to include("out of stock")
      end
    end
  end
end
```

## Model Testing

### Test Constraints and Complex Logic
```ruby
RSpec.describe Product do
  describe "database constraints" do
    it "enforces positive price" do
      product = build(:product, price: -10)
      
      expect { product.save(validate: false) }
        .to raise_error(ActiveRecord::StatementInvalid)
    end
  end
  
  describe "complex scopes" do
    it "returns products needing reorder" do
      create(:product, quantity: 5, reorder_level: 10)  # Needs reorder
      create(:product, quantity: 15, reorder_level: 10) # Doesn't need
      
      expect(Product.needing_reorder.count).to eq(1)
    end
  end
  
  describe "business logic" do
    it "calculates days until out of stock" do
      product = create(:product, 
        quantity: 100,
        daily_sales_average: 10
      )
      
      expect(product.days_until_out_of_stock).to eq(10)
    end
  end
end
```

## API Testing

### Test Contracts and Edge Cases
```ruby
RSpec.describe "API V1 Products" do
  describe "GET /api/v1/products" do
    it "returns paginated products" do
      create_list(:product, 30)
      
      get "/api/v1/products", params: { page: 2, per_page: 10 }
      
      json = JSON.parse(response.body)
      
      expect(response).to have_http_status(:ok)
      expect(json['data'].size).to eq(10)
      expect(json['meta']['current_page']).to eq(2)
    end
    
    it "filters by category" do
      hardware = create(:category, name: "Hardware")
      create(:product, category: hardware)
      create(:product, category: create(:category))
      
      get "/api/v1/products", params: { category_id: hardware.id }
      
      json = JSON.parse(response.body)
      expect(json['data'].size).to eq(1)
    end
  end
  
  describe "POST /api/v1/products" do
    it "creates product with valid params" do
      params = { product: { name: "Widget", price: 29.99 } }
      
      post "/api/v1/products", 
           params: params,
           headers: { "Authorization" => "Bearer #{token}" }
      
      expect(response).to have_http_status(:created)
      expect(Product.last.name).to eq("Widget")
    end
    
    it "returns errors for invalid params" do
      params = { product: { name: "", price: -10 } }
      
      post "/api/v1/products",
           params: params,
           headers: { "Authorization" => "Bearer #{token}" }
      
      expect(response).to have_http_status(:unprocessable_entity)
      
      json = JSON.parse(response.body)
      expect(json['errors']).to be_present
    end
  end
end
```

## Test Helpers

### Create Diagnostic Helpers
```ruby
# spec/support/with_clues.rb
def with_clues(&block)
  yield
rescue RSpec::Expectations::ExpectationNotMetError => e
  puts "\n===== DIAGNOSTIC CLUES ====="
  puts "Current URL: #{current_url}" if defined?(current_url)
  puts "Page HTML:\n#{page.html}" if defined?(page)
  puts "========================\n"
  raise e
end

# Usage in tests
it "completes checkout" do
  with_clues do
    visit checkout_path
    fill_in "Card Number", with: "4242424242424242"
    click_button "Pay"
    
    expect(page).to have_content("Order confirmed")
  end
end
```

### Use Factories Effectively
```ruby
# spec/factories/users.rb
FactoryBot.define do
  factory :user do
    sequence(:email) { |n| "user#{n}@example.com" }
    password { "password123" }
    
    trait :admin do
      role { "admin" }
    end
    
    trait :with_subscription do
      after(:create) do |user|
        create(:subscription, user: user)
      end
    end
  end
end

# Usage
create(:user)
create(:user, :admin)
create(:user, :with_subscription)
```

## Test Data Management

### Use Database Cleaner Appropriately
```ruby
# spec/rails_helper.rb
RSpec.configure do |config|
  config.before(:suite) do
    DatabaseCleaner.strategy = :transaction
    DatabaseCleaner.clean_with(:truncation)
  end
  
  config.before(:each) do
    DatabaseCleaner.start
  end
  
  config.after(:each) do
    DatabaseCleaner.clean
  end
  
  # Use truncation for system tests
  config.before(:each, type: :system) do
    DatabaseCleaner.strategy = :truncation
  end
end
```

## Performance Considerations

### Run Tests in Parallel
```ruby
# .rspec_parallel
--format progress
--format ParallelTests::RSpec::RuntimeLogger
--out tmp/parallel_runtime_rspec.log

# Run with: bundle exec parallel_rspec spec/
```

### Use Spring for Faster Tests
```ruby
# Gemfile
group :development, :test do
  gem 'spring-commands-rspec'
end

# Run with: spring rspec spec/models/user_spec.rb
```

## Best Practices Summary

1. **Test behavior, not implementation**
2. **Focus on critical paths and complex logic**
3. **Use appropriate test types for each scenario**
4. **Keep tests fast and reliable**
5. **Make tests readable and maintainable**
6. **Use factories and fixtures appropriately**
7. **Test edge cases and error conditions**
8. **Avoid testing framework features**