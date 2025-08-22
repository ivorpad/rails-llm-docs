# View Architecture Best Practices

## Core Principles
Use semantic HTML, expose one instance variable per action, name variables after resources, and keep logic out of views.

## Semantic HTML

### Build Views with Meaningful Tags
```erb
<!-- BAD - Divs everywhere -->
<div class="product">
  <div class="title"><%= product.name %></div>
  <div class="description"><%= product.description %></div>
  <div class="price">$<%= product.price %></div>
</div>

<!-- GOOD - Semantic HTML -->
<article class="product">
  <header>
    <h2><%= product.name %></h2>
  </header>
  <section class="description">
    <p><%= product.description %></p>
  </section>
  <footer>
    <span class="price">$<%= product.price %></span>
  </footer>
</article>
```

### Use Appropriate HTML5 Elements
```erb
<!-- Navigation -->
<nav aria-label="Main navigation">
  <ul>
    <li><%= link_to "Home", root_path %></li>
    <li><%= link_to "Products", products_path %></li>
    <li><%= link_to "About", about_path %></li>
  </ul>
</nav>

<!-- Main content -->
<main>
  <article>
    <header>
      <h1><%= @article.title %></h1>
      <time datetime="<%= @article.published_at.iso8601 %>">
        <%= @article.published_at.strftime("%B %d, %Y") %>
      </time>
    </header>
    
    <section>
      <%= @article.content %>
    </section>
    
    <aside>
      <h3>Related Articles</h3>
      <!-- Related content -->
    </aside>
  </article>
</main>

<!-- Footer -->
<footer>
  <address>
    Contact us at <a href="mailto:info@example.com">info@example.com</a>
  </address>
</footer>
```

## One Instance Variable Per Action

### Single Resource Pattern
```ruby
# Controller
class ProductsController < ApplicationController
  def show
    @product = Product.find(params[:id])
  end
  
  def index
    @products = Product.page(params[:page])
  end
  
  def new
    @product = Product.new
  end
end
```

### Name Variables After Resources
```ruby
# GOOD - Clear resource naming
class OrdersController < ApplicationController
  def show
    @order = Order.find(params[:id])  # Not @data or @obj
  end
  
  def index
    @orders = Order.recent  # Plural for collections
  end
end

# In views
<h1>Order #<%= @order.number %></h1>

<% @orders.each do |order| %>
  <%= render order %>
<% end %>
```

### Handling Reference Data
```ruby
# ACCEPTABLE - Reference data as additional variables
class ProductsController < ApplicationController
  def new
    @product = Product.new
    @categories = Category.active  # Reference data
  end
  
  def edit
    @product = Product.find(params[:id])
    @categories = Category.active  # Reference data
  end
end

# BETTER - Extract to helper method
class ProductsController < ApplicationController
  helper_method :categories
  
  def new
    @product = Product.new
  end
  
  private
  
  def categories
    @categories ||= Category.active
  end
end

# In view
<%= form.select :category_id, 
                options_from_collection_for_select(categories, :id, :name) %>
```

## Keep Logic Out of Views

### BAD: Logic in Views
```erb
<!-- BAD - Business logic in view -->
<% if @order.total > 100 && @order.user.member_since < 1.year.ago %>
  <div class="discount">
    You qualify for a <%= (@order.total * 0.1).round(2) %> discount!
  </div>
<% end %>

<% if @product.quantity <= @product.reorder_level && 
      @product.quantity > 0 %>
  <span class="low-stock">Only <%= @product.quantity %> left!</span>
<% elsif @product.quantity == 0 %>
  <span class="out-of-stock">Out of Stock</span>
<% end %>
```

### GOOD: Logic in Models/Helpers
```ruby
# Model
class Order < ApplicationRecord
  def qualifies_for_discount?
    total > 100 && user.member_for_over_a_year?
  end
  
  def discount_amount
    (total * 0.1).round(2)
  end
end

class Product < ApplicationRecord
  def stock_status
    return :out_of_stock if quantity == 0
    return :low_stock if quantity <= reorder_level
    :in_stock
  end
  
  def stock_message
    case stock_status
    when :out_of_stock then "Out of Stock"
    when :low_stock then "Only #{quantity} left!"
    else "In Stock"
    end
  end
end
```

```erb
<!-- Clean view -->
<% if @order.qualifies_for_discount? %>
  <div class="discount">
    You qualify for a $<%= @order.discount_amount %> discount!
  </div>
<% end %>

<span class="stock-<%= @product.stock_status %>">
  <%= @product.stock_message %>
</span>
```

## Layout Architecture

### Application Layout
```erb
<!-- app/views/layouts/application.html.erb -->
<!DOCTYPE html>
<html lang="<%= I18n.locale %>">
  <head>
    <meta charset="utf-8">
    <meta name="viewport" content="width=device-width, initial-scale=1">
    <title><%= content_for(:title) || "My App" %></title>
    <%= csrf_meta_tags %>
    <%= csp_meta_tag %>
    
    <%= stylesheet_link_tag "application" %>
    <%= javascript_include_tag "application", defer: true %>
    
    <%= yield :head %>
  </head>
  
  <body class="<%= controller_name %> <%= action_name %>">
    <%= render "shared/header" %>
    
    <main id="main-content">
      <%= render "shared/flash_messages" %>
      <%= yield %>
    </main>
    
    <%= render "shared/footer" %>
    <%= yield :javascript %>
  </body>
</html>
```

### Content Blocks
```erb
<!-- In specific views -->
<% content_for :title do %>
  <%= @product.name %> - Products
<% end %>

<% content_for :head do %>
  <meta name="description" content="<%= @product.description %>">
  <meta property="og:title" content="<%= @product.name %>">
<% end %>

<% content_for :javascript do %>
  <script>
    // Page-specific JavaScript
  </script>
<% end %>
```

## Form Best Practices

### Use Rails Form Helpers
```erb
<%= form_with model: @product do |form| %>
  <% if @product.errors.any? %>
    <div class="error-messages">
      <h3><%= pluralize(@product.errors.count, "error") %> prevented saving:</h3>
      <ul>
        <% @product.errors.full_messages.each do |message| %>
          <li><%= message %></li>
        <% end %>
      </ul>
    </div>
  <% end %>
  
  <div class="field">
    <%= form.label :name %>
    <%= form.text_field :name, required: true %>
  </div>
  
  <div class="field">
    <%= form.label :category_id %>
    <%= form.collection_select :category_id, 
                                Category.active, 
                                :id, 
                                :name,
                                { prompt: "Select a category" },
                                { required: true } %>
  </div>
  
  <div class="field">
    <%= form.label :description %>
    <%= form.text_area :description, rows: 5 %>
  </div>
  
  <div class="actions">
    <%= form.submit %>
  </div>
<% end %>
```

## Link and URL Helpers

### Always Use Path Helpers
```erb
<!-- BAD - Hardcoded URLs -->
<a href="/products">Products</a>
<a href="/products/<%= @product.id %>">View Product</a>

<!-- GOOD - Path helpers -->
<%= link_to "Products", products_path %>
<%= link_to "View Product", product_path(@product) %>
<%= link_to "Edit", edit_product_path(@product) %>

<!-- With nested routes -->
<%= link_to "Comments", product_comments_path(@product) %>

<!-- With parameters -->
<%= link_to "Filtered Products", 
            products_path(category: "electronics", sort: "price") %>
```

## Conditional Rendering

### Clean Conditionals
```erb
<!-- Use presence checks -->
<% if @products.any? %>
  <div class="products-grid">
    <%= render @products %>
  </div>
<% else %>
  <div class="empty-state">
    <p>No products found.</p>
    <%= link_to "Add a product", new_product_path %>
  </div>
<% end %>

<!-- Use unless for negative conditions -->
<% unless current_user.verified? %>
  <div class="alert">
    Please verify your email address.
  </div>
<% end %>

<!-- Use ternary for simple cases -->
<div class="status <%= @order.paid? ? 'paid' : 'pending' %>">
  <%= @order.paid? ? 'Paid' : 'Payment Pending' %>
</div>
```

## Avoiding N+1 Queries in Views

### Use Includes in Controllers
```ruby
# Controller
class ProductsController < ApplicationController
  def index
    # BAD - Will cause N+1 queries in view
    @products = Product.all
    
    # GOOD - Eager load associations
    @products = Product.includes(:category, :reviews, :images)
  end
end
```

```erb
<!-- This won't cause N+1 with includes -->
<% @products.each do |product| %>
  <div class="product">
    <h3><%= product.name %></h3>
    <p>Category: <%= product.category.name %></p>
    <p>Reviews: <%= product.reviews.count %></p>
    <div class="images">
      <% product.images.each do |image| %>
        <%= image_tag image.url %>
      <% end %>
    </div>
  </div>
<% end %>
```

## Best Practices Summary

1. **Use semantic HTML for structure**
2. **One instance variable per action**
3. **Name variables after resources**
4. **Keep business logic out of views**
5. **Use Rails helpers for forms and links**
6. **Handle conditionals cleanly**
7. **Prevent N+1 queries with includes**
8. **Use content_for for layout flexibility**