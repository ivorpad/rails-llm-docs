# Nested Routes Best Practices

## Core Principle
Use nested routes strategically. They should represent true parent-child relationships, not just associations.

## When to Use Nested Routes

### Good Use Cases
```ruby
# True ownership relationship
resources :users do
  resources :addresses  # User owns addresses
end

# Scoped collections
resources :projects do
  resources :tasks  # Tasks belong to specific project
end

# Sub-resources that don't exist independently
resources :orders do
  resources :line_items  # Line items only exist within orders
end
```

### When NOT to Nest
```ruby
# BAD - Deep nesting
resources :companies do
  resources :departments do
    resources :employees do
      resources :tasks
    end
  end
end
# URL: /companies/1/departments/2/employees/3/tasks/4 

# GOOD - Shallow nesting
resources :companies do
  resources :departments, shallow: true
end
resources :departments do
  resources :employees, shallow: true
end
resources :employees do
  resources :tasks, shallow: true
end
# URLs:
# /companies/1/departments (index, create)
# /departments/2 (show, update, destroy)
# /employees/3/tasks (index, create)
# /tasks/4 (show, update, destroy)
```

## Shallow Nesting

### Use Shallow Routes for Better URLs
```ruby
# Using shallow option
resources :articles do
  resources :comments, shallow: true
end

# Generates:
# GET    /articles/:article_id/comments     -> comments#index
# POST   /articles/:article_id/comments     -> comments#create
# GET    /comments/:id                       -> comments#show
# GET    /comments/:id/edit                  -> comments#edit
# PATCH  /comments/:id                       -> comments#update
# DELETE /comments/:id                       -> comments#destroy

# Controller
class CommentsController < ApplicationController
  before_action :set_article, only: [:index, :create]
  before_action :set_comment, only: [:show, :edit, :update, :destroy]
  
  def index
    @comments = @article.comments
  end
  
  def create
    @comment = @article.comments.build(comment_params)
    if @comment.save
      redirect_to @comment
    else
      render :new
    end
  end
  
  def show
    # @comment already set
  end
  
  private
  
  def set_article
    @article = Article.find(params[:article_id])
  end
  
  def set_comment
    @comment = Comment.find(params[:id])
  end
end
```

## Alternative to Deep Nesting

### Use Separate Resources
```ruby
# Instead of deep nesting
# BAD
resources :organizations do
  resources :projects do
    resources :tasks do
      resources :comments
    end
  end
end

# GOOD - Separate resources with filters
resources :organizations
resources :projects
resources :tasks
resources :comments

# In controllers, use scoping
class TasksController < ApplicationController
  def index
    @tasks = Task.all
    @tasks = @tasks.where(project_id: params[:project_id]) if params[:project_id]
    @tasks = @tasks.where(organization_id: params[:organization_id]) if params[:organization_id]
  end
end
```

### Use Namespacing for Organization
```ruby
# Organize related resources
namespace :project do
  resources :tasks
  resources :milestones
  resources :members
end

# Results in:
# /project/tasks
# /project/milestones
# /project/members

# With admin section
namespace :admin do
  resources :projects do
    resources :settings, only: [:edit, :update]
  end
end
```

## Member and Collection Routes

### Use Sparingly
```ruby
# AVOID custom actions when possible
resources :projects do
  member do
    post :archive  # Consider: resource :archive
    post :restore  # Consider: resource :restoration
  end
  
  collection do
    get :archived  # This is acceptable for filtering
  end
end

# BETTER - Create resources
resources :projects do
  resource :archive, only: [:create, :destroy]
  collection do
    get :archived
  end
end
```

## Concerns for Shared Behavior

### Extract Common Patterns
```ruby
# config/routes.rb
concern :commentable do
  resources :comments, shallow: true
end

concern :archivable do
  resource :archive, only: [:create, :destroy]
end

resources :articles, concerns: [:commentable, :archivable]
resources :photos, concerns: [:commentable]
resources :projects, concerns: [:archivable]
```

## Route Constraints

### Add Constraints for IDs
```ruby
# Ensure IDs are numeric
resources :users, constraints: { id: /\d+/ } do
  resources :posts
end

# Custom constraints
class SubdomainConstraint
  def self.matches?(request)
    request.subdomain.present? && request.subdomain != 'www'
  end
end

constraints SubdomainConstraint do
  resources :accounts
end
```

## URL Helpers with Nested Routes

### Use Path Helpers Effectively
```ruby
# With nested routes
resources :magazines do
  resources :articles
end

# In views and controllers
magazine_articles_path(@magazine)     # /magazines/1/articles
magazine_article_path(@magazine, @article)  # /magazines/1/articles/2
new_magazine_article_path(@magazine)  # /magazines/1/articles/new

# With shallow nesting
resources :magazines do
  resources :articles, shallow: true
end

# Simpler helpers for non-nested actions
article_path(@article)  # /articles/2
edit_article_path(@article)  # /articles/2/edit
```

## Controller Organization

### Handle Nested Resources Properly
```ruby
class ArticlesController < ApplicationController
  before_action :set_magazine, only: [:index, :new, :create]
  before_action :set_article, only: [:show, :edit, :update, :destroy]
  
  def index
    @articles = if @magazine
                  @magazine.articles
                else
                  Article.all
                end
  end
  
  def new
    @article = if @magazine
                 @magazine.articles.build
               else
                 Article.new
               end
  end
  
  def create
    @article = if @magazine
                 @magazine.articles.build(article_params)
               else
                 Article.new(article_params)
               end
    
    if @article.save
      redirect_to @article
    else
      render :new
    end
  end
  
  private
  
  def set_magazine
    @magazine = Magazine.find(params[:magazine_id]) if params[:magazine_id]
  end
  
  def set_article
    @article = Article.find(params[:id])
  end
end
```

## Best Practices for APIs

### Keep API Routes Simple
```ruby
namespace :api do
  namespace :v1 do
    # Avoid deep nesting in APIs
    resources :users, only: [:index, :show] do
      # Only nest for creation/listing
      resources :orders, only: [:index, :create]
    end
    
    # Access orders directly for other operations
    resources :orders, only: [:show, :update, :destroy]
  end
end
```

## Testing Nested Routes

### Test Route Generation
```ruby
RSpec.describe "Routes" do
  it "routes to nested comments" do
    expect(get: "/posts/1/comments").to route_to(
      controller: "comments",
      action: "index",
      post_id: "1"
    )
  end
  
  it "routes to comment directly with shallow nesting" do
    expect(get: "/comments/1").to route_to(
      controller: "comments",
      action: "show",
      id: "1"
    )
  end
end
```

## Summary

### Guidelines
1. **Never nest more than one level deep**
2. **Use shallow nesting for better URLs**
3. **Nest only when there's true ownership**
4. **Consider separate resources with filters instead**
5. **Use concerns for repeated patterns**
6. **Keep API routes simple**
7. **Test your route generation**

### Decision Tree
- Does the child resource make sense without the parent? → **Don't nest**
- Do you need the parent ID for all actions? → **Consider nesting**
- Is the nesting more than one level? → **Use shallow or separate**
- Is this just for organization? → **Use namespacing instead**