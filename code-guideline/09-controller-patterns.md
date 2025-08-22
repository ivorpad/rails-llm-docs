# Controller Patterns

## Core Principle
Controllers are configuration, not implementation. They should only handle HTTP concerns and delegate business logic to services.

## Keep Controllers Thin

### Controllers Should Only:
1. Parse and validate parameters
2. Call service objects
3. Handle the response
4. Set instance variables for views

```ruby
# BAD - Business logic in controller
class OrdersController < ApplicationController
  def create
    @order = Order.new(order_params)
    
    if @order.save
      @order.line_items.each do |item|
        product = item.product
        product.decrement!(:quantity, item.quantity)
        
        if product.quantity <= product.reorder_level
          InventoryMailer.low_stock(product).deliver_later
        end
      end
      
      PaymentService.charge(@order.total, params[:payment_token])
      OrderMailer.confirmation(@order).deliver_later
      
      redirect_to @order
    else
      render :new
    end
  end
end

# GOOD - Thin controller
class OrdersController < ApplicationController
  def create
    result = CreateOrder.new.call(
      order_params: order_params,
      payment_token: params[:payment_token]
    )
    
    if result.success?
      redirect_to result.order
    else
      @order = result.order
      @errors = result.errors
      render :new
    end
  end
end
```

## Parameter Handling

### Convert Parameters to Rich Types
```ruby
class EventsController < ApplicationController
  def index
    # BAD - Using strings directly
    @events = Event.where('starts_at >= ?', params[:date])
    
    # GOOD - Convert to proper type
    @events = Event.where('starts_at >= ?', parsed_date)
  end
  
  private
  
  def parsed_date
    return Date.current unless params[:date].present?
    
    Date.parse(params[:date])
  rescue ArgumentError
    Date.current
  end
end
```

### Strong Parameters
```ruby
class UsersController < ApplicationController
  def create
    # Simple parameters
    @user = User.new(user_params)
    
    if @user.save
      redirect_to @user
    else
      render :new
    end
  end
  
  private
  
  def user_params
    params.require(:user).permit(:name, :email, :password)
  end
end

# Complex nested parameters
class OrdersController < ApplicationController
  private
  
  def order_params
    params.require(:order).permit(
      :customer_id,
      :shipping_address,
      line_items_attributes: [:id, :product_id, :quantity, :_destroy],
      payment_attributes: [:method, :token]
    )
  end
end
```

## Callback Usage

### Use Callbacks Sparingly
```ruby
# GOOD - Minimal callbacks for cross-cutting concerns
class ApplicationController < ActionController::Base
  before_action :authenticate_user!
  before_action :set_locale
  
  private
  
  def set_locale
    I18n.locale = params[:locale] || I18n.default_locale
  end
end

# BAD - Too many callbacks
class ProductsController < ApplicationController
  before_action :load_categories
  before_action :load_brands  
  before_action :check_inventory
  before_action :verify_pricing
  before_action :set_product, only: [:show, :edit, :update, :destroy]
  before_action :check_permissions, only: [:edit, :update, :destroy]
  after_action :log_activity
  after_action :update_cache
  
  # Controller actions become unclear
end

# BETTER - Explicit method calls
class ProductsController < ApplicationController
  def index
    @categories = Category.active
    @products = FilterProducts.new.call(
      products: Product.all,
      filters: filter_params
    )
  end
  
  def show
    @product = Product.find(params[:id])
    @related_products = @product.related_products
  end
end
```

## Response Handling

### Consistent Response Patterns
```ruby
class UsersController < ApplicationController
  def create
    result = CreateUser.new.call(user_params)
    
    respond_to do |format|
      if result.success?
        format.html { redirect_to result.user, notice: 'User created' }
        format.json { render json: result.user, status: :created }
      else
        format.html { 
          @user = result.user
          render :new 
        }
        format.json { 
          render json: { errors: result.errors }, status: :unprocessable_entity 
        }
      end
    end
  end
end
```

## Error Handling

### Rescue Specific Exceptions
```ruby
class ApplicationController < ActionController::Base
  # Handle record not found
  rescue_from ActiveRecord::RecordNotFound do |exception|
    respond_to do |format|
      format.html { render file: 'public/404.html', status: :not_found }
      format.json { render json: { error: 'Not found' }, status: :not_found }
    end
  end
  
  # Handle authorization errors
  rescue_from Pundit::NotAuthorizedError do |exception|
    respond_to do |format|
      format.html { 
        redirect_to root_path, alert: 'Not authorized' 
      }
      format.json { 
        render json: { error: 'Not authorized' }, status: :forbidden 
      }
    end
  end
end

class PaymentsController < ApplicationController
  rescue_from PaymentGateway::Error do |exception|
    redirect_to checkout_path, alert: exception.message
  end
  
  def create
    # Payment processing that might raise PaymentGateway::Error
  end
end
```

## Instance Variables

### One Resource Per Action
```ruby
# GOOD - Single, clearly named instance variable
class ProductsController < ApplicationController
  def show
    @product = Product.find(params[:id])
  end
  
  def index
    @products = Product.page(params[:page])
  end
end

# ACCEPTABLE - Additional reference data
class ProductsController < ApplicationController
  def new
    @product = Product.new
    @categories = Category.active  # Reference data
  end
end

# BAD - Too many instance variables
class ProductsController < ApplicationController
  def show
    @product = Product.find(params[:id])
    @reviews = @product.reviews
    @related = @product.related_products
    @recently_viewed = session[:recently_viewed]
    @cart_items = current_cart.items
    @user_wishlist = current_user.wishlist_items
  end
end
```

## Controller Concerns

### Extract Shared Behavior
```ruby
# app/controllers/concerns/pagination.rb
module Pagination
  extend ActiveSupport::Concern
  
  included do
    helper_method :page, :per_page
  end
  
  def page
    params[:page] || 1
  end
  
  def per_page
    params[:per_page] || 25
  end
  
  def paginate(relation)
    relation.page(page).per(per_page)
  end
end

# app/controllers/products_controller.rb
class ProductsController < ApplicationController
  include Pagination
  
  def index
    @products = paginate(Product.all)
  end
end
```

## API Controllers

### Separate API Controllers
```ruby
# app/controllers/api/v1/base_controller.rb
module Api
  module V1
    class BaseController < ActionController::API
      before_action :authenticate_api_user!
      
      private
      
      def authenticate_api_user!
        token = request.headers['Authorization']
        @current_user = User.find_by_api_token(token)
        
        render json: { error: 'Unauthorized' }, status: :unauthorized unless @current_user
      end
    end
  end
end

# app/controllers/api/v1/products_controller.rb
module Api
  module V1
    class ProductsController < BaseController
      def index
        products = Product.active
        render json: products
      end
      
      def show
        product = Product.find(params[:id])
        render json: product
      rescue ActiveRecord::RecordNotFound
        render json: { error: 'Not found' }, status: :not_found
      end
    end
  end
end
```

## Testing Controllers

### Focus on Integration, Not Units
```ruby
# Don't over-test controllers
RSpec.describe ProductsController do
  describe "GET #index" do
    it "returns products" do
      products = create_list(:product, 3)
      
      get :index
      
      expect(response).to be_successful
      expect(assigns(:products)).to match_array(products)
    end
  end
end

# Better - Test the full stack with system tests
RSpec.describe "Products", type: :system do
  it "displays products" do
    products = create_list(:product, 3)
    
    visit products_path
    
    products.each do |product|
      expect(page).to have_content(product.name)
    end
  end
end
```