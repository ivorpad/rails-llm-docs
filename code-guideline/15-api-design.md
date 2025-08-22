# API Design Best Practices

## Core Principles
Be clear about what and who your API is for. Write APIs the same way you write other Rails code. Keep it simple and consistent.

## API Structure

### Use Standard REST Patterns
```ruby
# routes.rb
namespace :api do
  namespace :v1 do
    resources :products, only: [:index, :show, :create, :update, :destroy]
    resources :orders do
      resources :line_items, only: [:index, :create, :destroy]
    end
  end
end
```

### Separate API Controllers
```ruby
# app/controllers/api/v1/base_controller.rb
module Api
  module V1
    class BaseController < ActionController::API
      include Pagination
      include ErrorHandling
      
      before_action :authenticate_request
      
      private
      
      def authenticate_request
        # Authentication logic
      end
    end
  end
end

# app/controllers/api/v1/products_controller.rb
module Api
  module V1
    class ProductsController < BaseController
      def index
        products = Product.active.page(params[:page])
        render json: products
      end
      
      def show
        product = Product.find(params[:id])
        render json: product
      end
      
      def create
        result = CreateProduct.new.call(product_params: product_params)
        
        if result.success?
          render json: result.product, status: :created
        else
          render json: { errors: result.errors }, status: :unprocessable_entity
        end
      end
      
      private
      
      def product_params
        params.require(:product).permit(:name, :price, :description)
      end
    end
  end
end
```

## Versioning

### Put Version in the URL
```ruby
# Simple and clear
namespace :api do
  namespace :v1 do
    resources :products
  end
  
  namespace :v2 do
    resources :products  # New version with breaking changes
  end
end

# URLs:
# GET /api/v1/products
# GET /api/v2/products
```

### Avoid Header Versioning
```ruby
# BAD - Version in headers is harder to test and document
request.headers['API-Version'] = 'v2'

# GOOD - Version in URL is explicit
GET /api/v2/products
```

## Response Format

### Consistent JSON Structure
```ruby
# Success response
{
  "data": {
    "id": 1,
    "type": "product",
    "attributes": {
      "name": "Widget",
      "price": 29.99
    }
  }
}

# Error response
{
  "errors": [
    {
      "status": "422",
      "title": "Validation Error",
      "detail": "Price must be greater than 0",
      "source": { "pointer": "/data/attributes/price" }
    }
  ]
}

# Collection response
{
  "data": [...],
  "meta": {
    "total": 100,
    "page": 1,
    "per_page": 25
  }
}
```

### Always Use Top-Level Keys
```ruby
# BAD - Array at root level
[
  { "id": 1, "name": "Product 1" },
  { "id": 2, "name": "Product 2" }
]

# GOOD - Wrapped in data key
{
  "data": [
    { "id": 1, "name": "Product 1" },
    { "id": 2, "name": "Product 2" }
  ]
}
```

## Status Codes

### Use Appropriate HTTP Status Codes
```ruby
class Api::V1::ProductsController < Api::V1::BaseController
  def create
    product = Product.new(product_params)
    
    if product.save
      render json: product, status: :created  # 201
    else
      render json: { errors: product.errors }, status: :unprocessable_entity  # 422
    end
  end
  
  def destroy
    product = Product.find(params[:id])
    product.destroy
    head :no_content  # 204
  end
  
  def show
    product = Product.find_by(id: params[:id])
    
    if product
      render json: product, status: :ok  # 200
    else
      render json: { error: 'Not found' }, status: :not_found  # 404
    end
  end
end
```

## Pagination

### Implement Standard Pagination
```ruby
class Api::V1::ProductsController < Api::V1::BaseController
  def index
    products = Product.page(params[:page]).per(params[:per_page] || 25)
    
    render json: {
      data: products,
      meta: pagination_meta(products)
    }
  end
  
  private
  
  def pagination_meta(collection)
    {
      current_page: collection.current_page,
      next_page: collection.next_page,
      prev_page: collection.prev_page,
      total_pages: collection.total_pages,
      total_count: collection.total_count
    }
  end
end
```

## Filtering and Sorting

### Query Parameters for Filtering
```ruby
class Api::V1::ProductsController < Api::V1::BaseController
  def index
    products = Product.all
    products = filter_products(products)
    products = sort_products(products)
    products = products.page(params[:page])
    
    render json: products
  end
  
  private
  
  def filter_products(products)
    products = products.where(category_id: params[:category_id]) if params[:category_id]
    products = products.where('price >= ?', params[:min_price]) if params[:min_price]
    products = products.where('price <= ?', params[:max_price]) if params[:max_price]
    products
  end
  
  def sort_products(products)
    case params[:sort]
    when 'price_asc'
      products.order(price: :asc)
    when 'price_desc'
      products.order(price: :desc)
    when 'name'
      products.order(:name)
    else
      products.order(created_at: :desc)
    end
  end
end

# Usage:
# GET /api/v1/products?category_id=5&min_price=10&sort=price_asc
```

## Error Handling

### Consistent Error Responses
```ruby
module Api
  module V1
    module ErrorHandling
      extend ActiveSupport::Concern
      
      included do
        rescue_from ActiveRecord::RecordNotFound, with: :not_found
        rescue_from ActiveRecord::RecordInvalid, with: :unprocessable_entity
        rescue_from ActionController::ParameterMissing, with: :bad_request
      end
      
      private
      
      def not_found(exception)
        render json: {
          errors: [{
            status: '404',
            title: 'Not Found',
            detail: exception.message
          }]
        }, status: :not_found
      end
      
      def unprocessable_entity(exception)
        render json: {
          errors: exception.record.errors.map do |error|
            {
              status: '422',
              title: 'Validation Error',
              detail: error.full_message,
              source: { pointer: "/data/attributes/#{error.attribute}" }
            }
          end
        }, status: :unprocessable_entity
      end
      
      def bad_request(exception)
        render json: {
          errors: [{
            status: '400',
            title: 'Bad Request',
            detail: exception.message
          }]
        }, status: :bad_request
      end
    end
  end
end
```

## Rate Limiting

### Implement Rate Limiting
```ruby
# Using rack-attack gem
class Rack::Attack
  throttle('api/ip', limit: 100, period: 1.minute) do |req|
    req.ip if req.path.start_with?('/api')
  end
  
  throttle('api/user', limit: 1000, period: 1.hour) do |req|
    if req.path.start_with?('/api') && req.env['api.user']
      req.env['api.user'].id
    end
  end
end

# Return rate limit info in headers
class Api::V1::BaseController < ActionController::API
  after_action :set_rate_limit_headers
  
  private
  
  def set_rate_limit_headers
    response.headers['X-RateLimit-Limit'] = '100'
    response.headers['X-RateLimit-Remaining'] = calculate_remaining
    response.headers['X-RateLimit-Reset'] = next_reset_time.to_i.to_s
  end
end
```

## Documentation

### Self-Documenting Endpoints
```ruby
# app/controllers/api/v1/documentation_controller.rb
class Api::V1::DocumentationController < Api::V1::BaseController
  skip_before_action :authenticate_request
  
  def index
    render json: {
      version: 'v1',
      endpoints: {
        products: {
          index: 'GET /api/v1/products',
          show: 'GET /api/v1/products/:id',
          create: 'POST /api/v1/products',
          update: 'PATCH /api/v1/products/:id',
          destroy: 'DELETE /api/v1/products/:id'
        }
      }
    }
  end
end
```