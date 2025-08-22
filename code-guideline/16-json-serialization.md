# JSON Serialization Best Practices

## Core Principle
Use `to_json` and Rails' built-in serialization. Keep it simple - avoid complex serialization libraries unless truly needed.

## Basic JSON Serialization

### Using to_json Directly
```ruby
class ProductsController < ApplicationController
  def show
    product = Product.find(params[:id])
    render json: product.to_json
  end
  
  def index
    products = Product.includes(:category)
    render json: products.to_json(include: :category)
  end
end
```

### Customizing with as_json
```ruby
class Product < ApplicationRecord
  def as_json(options = {})
    super(options.merge(
      only: [:id, :name, :price, :description],
      include: {
        category: { only: [:id, :name] }
      },
      methods: [:availability_status, :discount_percentage]
    ))
  end
  
  def availability_status
    if quantity > 10
      "in_stock"
    elsif quantity > 0
      "low_stock"
    else
      "out_of_stock"
    end
  end
  
  def discount_percentage
    return 0 unless on_sale?
    ((original_price - price) / original_price * 100).round
  end
end
```

## Always Use Top-Level Keys

### Wrap Responses Consistently
```ruby
class Api::V1::ProductsController < Api::V1::BaseController
  def index
    products = Product.page(params[:page])
    
    render json: {
      data: products.as_json,
      meta: {
        current_page: products.current_page,
        total_pages: products.total_pages,
        total_count: products.total_count
      }
    }
  end
  
  def show
    product = Product.find(params[:id])
    
    render json: {
      data: product.as_json
    }
  end
  
  def create
    product = Product.new(product_params)
    
    if product.save
      render json: {
        data: product.as_json,
        message: "Product created successfully"
      }, status: :created
    else
      render json: {
        errors: product.errors.as_json,
        message: "Validation failed"
      }, status: :unprocessable_entity
    end
  end
end
```

## Custom Serialization in Models

### Per-Model Serialization
```ruby
class User < ApplicationRecord
  # Different serialization for different contexts
  def as_json(options = {})
    if options[:context] == :public
      public_json
    elsif options[:context] == :admin
      admin_json
    else
      default_json
    end
  end
  
  private
  
  def default_json
    {
      id: id,
      name: name,
      email: email,
      created_at: created_at
    }
  end
  
  def public_json
    {
      id: id,
      name: name,
      avatar_url: avatar_url
    }
  end
  
  def admin_json
    {
      id: id,
      name: name,
      email: email,
      role: role,
      last_login_at: last_login_at,
      created_at: created_at,
      updated_at: updated_at
    }
  end
end

# Usage in controller
render json: @user.as_json(context: :admin)
```

### Handling Associations
```ruby
class Order < ApplicationRecord
  has_many :line_items
  belongs_to :user
  
  def as_json(options = {})
    {
      id: id,
      number: number,
      status: status,
      total: total,
      created_at: created_at,
      user: user_summary,
      line_items: line_items_summary
    }
  end
  
  private
  
  def user_summary
    {
      id: user.id,
      name: user.name,
      email: user.email
    }
  end
  
  def line_items_summary
    line_items.map do |item|
      {
        id: item.id,
        product_name: item.product.name,
        quantity: item.quantity,
        price: item.price,
        subtotal: item.subtotal
      }
    end
  end
end
```

## Serialization Concerns

### Reusable Serialization Logic
```ruby
# app/models/concerns/api_serializable.rb
module ApiSerializable
  extend ActiveSupport::Concern
  
  included do
    def as_json(options = {})
      options[:except] ||= []
      options[:except] += [:created_at, :updated_at] unless options[:include_timestamps]
      
      result = super(options)
      
      # Add common metadata
      if options[:include_meta]
        result[:_meta] = {
          type: self.class.name.underscore,
          url: url_for(self)
        }
      end
      
      result
    end
  end
end

class Product < ApplicationRecord
  include ApiSerializable
  
  def as_json(options = {})
    super(options.merge(
      only: [:id, :name, :price],
      methods: [:availability_status]
    ))
  end
end
```

## Collection Serialization

### Efficient Collection Handling
```ruby
class ProductsController < ApplicationController
  def index
    products = Product.includes(:category, :images)
    
    render json: {
      data: serialize_collection(products),
      meta: pagination_meta(products)
    }
  end
  
  private
  
  def serialize_collection(products)
    products.map { |product| serialize_product(product) }
  end
  
  def serialize_product(product)
    {
      id: product.id,
      name: product.name,
      price: product.price,
      category: product.category.name,
      image_url: product.images.first&.url
    }
  end
  
  def pagination_meta(collection)
    {
      current_page: collection.current_page,
      per_page: collection.limit_value,
      total_pages: collection.total_pages,
      total_count: collection.total_count
    }
  end
end
```

## Conditional Attributes

### Include Attributes Based on Context
```ruby
class Api::V1::BaseController < ActionController::API
  private
  
  def current_ability
    @current_ability ||= Ability.new(current_user)
  end
  
  def serialize_with_permissions(object)
    json = object.as_json
    
    # Add permissions metadata
    json[:_permissions] = {
      can_edit: can?(:edit, object),
      can_delete: can?(:destroy, object)
    }
    
    # Include sensitive data only if authorized
    if can?(:view_sensitive_data, object)
      json[:sensitive_data] = object.sensitive_attributes
    end
    
    json
  end
end

class Api::V1::UsersController < Api::V1::BaseController
  def show
    user = User.find(params[:id])
    render json: serialize_with_permissions(user)
  end
end
```

## Performance Optimization

### Avoid N+1 in Serialization
```ruby
class OrdersController < ApplicationController
  def index
    # Eager load all needed associations
    orders = Order.includes(
      :user,
      line_items: [:product, :variant]
    )
    
    render json: orders.map { |order| serialize_order(order) }
  end
  
  private
  
  def serialize_order(order)
    {
      id: order.id,
      total: order.total,
      user_name: order.user.name,  # No N+1
      items: order.line_items.map { |item|
        {
          product_name: item.product.name,  # No N+1
          variant_name: item.variant&.name   # No N+1
        }
      }
    }
  end
end
```

### Cache Expensive Serialization
```ruby
class Product < ApplicationRecord
  def as_json(options = {})
    Rails.cache.fetch([cache_key_with_version, 'json']) do
      super(
        only: [:id, :name, :price],
        include: {
          category: { only: [:id, :name] },
          reviews: { only: [:id, :rating, :comment] }
        },
        methods: [:average_rating, :review_count]
      )
    end
  end
  
  def average_rating
    reviews.average(:rating)&.round(2)
  end
  
  def review_count
    reviews.count
  end
end
```

## Error Serialization

### Consistent Error Format
```ruby
class Api::V1::BaseController < ActionController::API
  rescue_from ActiveRecord::RecordInvalid do |exception|
    render json: serialize_errors(exception.record), 
           status: :unprocessable_entity
  end
  
  rescue_from ActiveRecord::RecordNotFound do |exception|
    render json: {
      errors: [{
        status: '404',
        title: 'Not Found',
        detail: exception.message
      }]
    }, status: :not_found
  end
  
  private
  
  def serialize_errors(record)
    {
      errors: record.errors.map do |error|
        {
          status: '422',
          source: { pointer: "/data/attributes/#{error.attribute}" },
          title: 'Validation Error',
          detail: error.full_message
        }
      end
    }
  end
end
```

## Alternative: Simple Serializer Classes

### When Models Get Complex
```ruby
# app/serializers/product_serializer.rb
class ProductSerializer
  def initialize(product, options = {})
    @product = product
    @options = options
  end
  
  def as_json
    {
      id: @product.id,
      name: @product.name,
      price: formatted_price,
      description: @product.description,
      images: images_json,
      category: category_json
    }.tap do |json|
      json[:inventory] = inventory_json if include_inventory?
    end
  end
  
  private
  
  attr_reader :product, :options
  
  def formatted_price
    "$%.2f" % product.price
  end
  
  def images_json
    product.images.map { |img| { url: img.url, alt: img.alt_text } }
  end
  
  def category_json
    return nil unless product.category
    {
      id: product.category.id,
      name: product.category.name
    }
  end
  
  def inventory_json
    {
      quantity: product.quantity,
      status: product.availability_status
    }
  end
  
  def include_inventory?
    options[:include_inventory] == true
  end
end

# Usage
render json: ProductSerializer.new(@product, include_inventory: true).as_json
```

## Best Practices Summary

1. **Use built-in Rails serialization**
2. **Always use top-level keys**
3. **Customize with as_json in models**
4. **Avoid N+1 queries with includes**
5. **Cache expensive serialization**
6. **Keep error format consistent**
7. **Consider serializer classes for complex cases**
8. **Don't use heavy serialization gems unless needed**