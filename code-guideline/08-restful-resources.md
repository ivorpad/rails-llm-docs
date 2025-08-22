# RESTful Resources Best Practices

## Core Principle
Don't create custom actions - create more resources. Every action in your application should map to a standard REST verb on a resource.

## The Seven Standard Actions
```ruby
# routes.rb
resources :products
# Creates:
# GET    /products          -> index
# GET    /products/new      -> new
# POST   /products          -> create
# GET    /products/:id      -> show
# GET    /products/:id/edit -> edit
# PATCH  /products/:id      -> update
# DELETE /products/:id      -> destroy
```

## Avoid Custom Actions

### BAD: Custom Actions
```ruby
# BAD - Custom actions on existing resources
resources :orders do
  member do
    post :cancel
    post :ship
    post :refund
    get :invoice
  end
end

# Results in non-RESTful routes:
# POST /orders/:id/cancel
# POST /orders/:id/ship
# POST /orders/:id/refund
# GET  /orders/:id/invoice
```

### GOOD: Create New Resources
```ruby
# GOOD - Each action becomes a resource
resources :orders do
  resource :cancellation, only: [:create]
  resource :shipment, only: [:create, :show]
  resource :refund, only: [:new, :create]
  resource :invoice, only: [:show]
end

# Results in RESTful routes:
# POST /orders/:order_id/cancellation
# POST /orders/:order_id/shipment
# GET  /orders/:order_id/shipment
# GET  /orders/:order_id/refund/new
# POST /orders/:order_id/refund
# GET  /orders/:order_id/invoice
```

## Resource-Oriented Thinking

### Example: User Account Activation
```ruby
# BAD - Custom action
resources :users do
  member do
    post :activate
    post :deactivate
  end
end

class UsersController < ApplicationController
  def activate
    @user = User.find(params[:id])
    @user.update!(active: true)
    redirect_to @user
  end
end

# GOOD - Activation as a resource
resources :users do
  resource :activation, only: [:create, :destroy]
end

class ActivationsController < ApplicationController
  before_action :set_user
  
  def create
    @user.activate!
    redirect_to @user, notice: 'User activated'
  end
  
  def destroy
    @user.deactivate!
    redirect_to @user, notice: 'User deactivated'
  end
  
  private
  
  def set_user
    @user = User.find(params[:user_id])
  end
end
```

### Example: Password Reset
```ruby
# BAD - Multiple custom actions
resources :users do
  collection do
    post :forgot_password
    post :reset_password
  end
end

# GOOD - Password reset as a resource
resources :password_resets, only: [:new, :create, :edit, :update]

class PasswordResetsController < ApplicationController
  def new
    # Show form to enter email
  end
  
  def create
    # Send reset email
    user = User.find_by(email: params[:email])
    user&.send_password_reset_email
    redirect_to login_path, notice: 'Check your email for reset instructions'
  end
  
  def edit
    # Show form to enter new password
    @user = User.find_by_reset_token!(params[:id])
  end
  
  def update
    # Update password
    @user = User.find_by_reset_token!(params[:id])
    if @user.reset_password(params[:password])
      redirect_to login_path, notice: 'Password updated'
    else
      render :edit
    end
  end
end
```

## Singular vs Plural Resources

### Use Singular Resources When Appropriate
```ruby
# For resources with only one instance per parent
resource :profile  # Current user's profile
resource :settings # Application settings
resource :subscription # User's single subscription

# Routes created:
# GET    /profile/new
# POST   /profile
# GET    /profile
# GET    /profile/edit
# PATCH  /profile
# DELETE /profile
# Note: No index action, no :id parameter
```

### Implementation Example
```ruby
# routes.rb
resource :profile, only: [:show, :edit, :update]

# profiles_controller.rb
class ProfilesController < ApplicationController
  before_action :require_login
  
  def show
    @profile = current_user.profile
  end
  
  def edit
    @profile = current_user.profile
  end
  
  def update
    @profile = current_user.profile
    if @profile.update(profile_params)
      redirect_to profile_path
    else
      render :edit
    end
  end
  
  private
  
  def profile_params
    params.require(:profile).permit(:bio, :avatar)
  end
end
```

## Search and Filters as Resources

### Search as a Resource
```ruby
# BAD - Search as custom action
resources :products do
  collection do
    get :search
  end
end

# GOOD - Search as a resource
resources :product_searches, only: [:new, :create, :show]

class ProductSearchesController < ApplicationController
  def new
    @search = ProductSearch.new
  end
  
  def create
    @search = ProductSearch.new(search_params)
    if @search.valid?
      redirect_to product_search_path(@search.to_param)
    else
      render :new
    end
  end
  
  def show
    @search = ProductSearch.from_param(params[:id])
    @products = @search.results
  end
end
```

## State Transitions as Resources

### Example: Order Workflow
```ruby
# Instead of custom actions for each state transition
resources :orders do
  namespace :admin do
    resources :approvals, only: [:create]
    resources :rejections, only: [:create]
    resources :fulfillments, only: [:create]
  end
end

# Controllers for each transition
module Admin
  class ApprovalsController < ApplicationController
    def create
      @order = Order.find(params[:order_id])
      result = ApproveOrder.new.call(order: @order)
      
      if result.success?
        redirect_to @order
      else
        redirect_to @order, alert: result.error
      end
    end
  end
end
```

## Collection Resources

### Bulk Operations as Resources
```ruby
# BAD - Custom bulk action
resources :emails do
  collection do
    post :mark_all_read
  end
end

# GOOD - Bulk operation as resource
resources :emails do
  resources :bulk_updates, only: [:create]
end

class BulkUpdatesController < ApplicationController
  def create
    email_ids = params[:email_ids]
    action = params[:action_type]
    
    case action
    when 'mark_read'
      Email.where(id: email_ids).update_all(read: true)
    when 'archive'
      Email.where(id: email_ids).update_all(archived: true)
    end
    
    redirect_to emails_path
  end
end
```

## Benefits of RESTful Resources

1. **Predictable URLs** - Developers know what to expect
2. **Simplified routing** - Standard patterns reduce complexity
3. **Better organization** - Each controller has single responsibility
4. **Easier testing** - Standard actions are easier to test
5. **API consistency** - REST principles translate well to APIs

## Route Constraints

### Use Constraints for Different Formats
```ruby
# Separate controllers for API and web
namespace :api do
  namespace :v1 do
    resources :products, only: [:index, :show]
  end
end

resources :products  # Web interface

# Or use constraints
resources :products, constraints: { format: :json } do
  # API-specific routes
end

resources :products, constraints: { format: :html } do
  # Web-specific routes
end
```