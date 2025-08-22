# Partials and Components Best Practices

## Core Principle
Use partials for simple re-use, View Components for complex UI logic. Always pass locals explicitly and use strict locals.

## Partials for Simple Re-use

### Basic Partial Usage
```erb
<!-- app/views/products/_product.html.erb -->
<article class="product">
  <h3><%= link_to product.name, product %></h3>
  <p><%= product.description %></p>
  <span class="price">$<%= product.price %></span>
</article>

<!-- Using the partial -->
<!-- app/views/products/index.html.erb -->
<div class="products">
  <%= render @products %>
</div>

<!-- Or explicitly -->
<%= render partial: "product", collection: @products %>

<!-- Single item -->
<%= render "product", product: @product %>
```

### Always Use Locals, Never Instance Variables
```erb
<!-- BAD - Partial uses instance variables -->
<!-- _header.html.erb -->
<header>
  <h1><%= @page_title %></h1>
  <p><%= @page_description %></p>
</header>

<!-- GOOD - Partial uses locals -->
<!-- _header.html.erb -->
<header>
  <h1><%= title %></h1>
  <p><%= description %></p>
</header>

<!-- Usage -->
<%= render "header", 
           title: "Products", 
           description: "Browse our catalog" %>
```

## Strict Locals (Rails 7.1+)

### Define Required and Optional Locals
```erb
<%# app/views/shared/_card.html.erb %>
<%# locals: (title:, body:, footer: nil, class_names: "") %>

<div class="card <%= class_names %>">
  <div class="card-header">
    <h3><%= title %></h3>
  </div>
  
  <div class="card-body">
    <%= body %>
  </div>
  
  <% if footer.present? %>
    <div class="card-footer">
      <%= footer %>
    </div>
  <% end %>
</div>

<!-- Usage -->
<%= render "shared/card",
           title: "Welcome",
           body: "Hello, world!" %>

<!-- With optional parameters -->
<%= render "shared/card",
           title: "Product Details",
           body: @product.description,
           footer: "Updated #{time_ago_in_words(@product.updated_at)} ago",
           class_names: "featured" %>
```

### Benefits of Strict Locals
```erb
<%# _user_info.html.erb %>
<%# locals: (user:, show_email: false, show_avatar: true) %>

<div class="user-info">
  <% if show_avatar %>
    <%= image_tag user.avatar_url, alt: user.name %>
  <% end %>
  
  <span class="name"><%= user.name %></span>
  
  <% if show_email %>
    <span class="email"><%= user.email %></span>
  <% end %>
</div>

<!-- Will raise error if required local is missing -->
<%= render "user_info" %>  <!-- Error: missing required local: user -->

<!-- Correct usage -->
<%= render "user_info", user: current_user %>
```

## View Components for Complex Logic

### When to Use View Components
- Complex conditionals
- Multiple related partials
- Need for testing
- Reusable across different views
- Encapsulated styling and behavior

### Creating a View Component
```ruby
# app/components/product_card_component.rb
class ProductCardComponent < ViewComponent::Base
  def initialize(product:, show_actions: true, featured: false)
    @product = product
    @show_actions = show_actions
    @featured = featured
  end
  
  private
  
  attr_reader :product, :show_actions, :featured
  
  def css_classes
    classes = ["product-card"]
    classes << "product-card--featured" if featured
    classes << "product-card--out-of-stock" unless product.in_stock?
    classes.join(" ")
  end
  
  def availability_badge
    if product.in_stock?
      content_tag(:span, "In Stock", class: "badge badge-success")
    else
      content_tag(:span, "Out of Stock", class: "badge badge-danger")
    end
  end
  
  def price_display
    if product.on_sale?
      content_tag(:span, class: "price") do
        concat content_tag(:s, number_to_currency(product.original_price))
        concat " "
        concat content_tag(:strong, number_to_currency(product.price))
      end
    else
      content_tag(:span, number_to_currency(product.price), class: "price")
    end
  end
end
```

### View Component Template
```erb
<!-- app/components/product_card_component.html.erb -->
<article class="<%= css_classes %>" data-product-id="<%= product.id %>">
  <% if featured %>
    <div class="featured-badge">Featured</div>
  <% end %>
  
  <header>
    <h3><%= link_to product.name, product %></h3>
    <%= availability_badge %>
  </header>
  
  <div class="product-body">
    <%= image_tag product.image_url, alt: product.name if product.image_url %>
    <p><%= truncate(product.description, length: 150) %></p>
  </div>
  
  <footer>
    <%= price_display %>
    
    <% if show_actions %>
      <div class="actions">
        <% if product.in_stock? %>
          <%= button_to "Add to Cart", 
                        cart_items_path, 
                        params: { product_id: product.id },
                        class: "btn btn-primary" %>
        <% else %>
          <%= button_to "Notify Me", 
                        product_notifications_path(product),
                        class: "btn btn-secondary" %>
        <% end %>
      </div>
    <% end %>
  </footer>
</article>
```

### Using View Components
```erb
<!-- In views -->
<%= render ProductCardComponent.new(product: @product) %>

<!-- With collection -->
<div class="products-grid">
  <% @products.each do |product| %>
    <%= render ProductCardComponent.new(
      product: product,
      show_actions: true,
      featured: product.featured?
    ) %>
  <% end %>
</div>
```

## Testing View Components

### Unit Testing Components
```ruby
# spec/components/product_card_component_spec.rb
RSpec.describe ProductCardComponent, type: :component do
  let(:product) { create(:product, price: 29.99) }
  
  it "renders product information" do
    render_inline(described_class.new(product: product))
    
    expect(page).to have_css("article.product-card")
    expect(page).to have_link(product.name)
    expect(page).to have_content("$29.99")
  end
  
  it "shows out of stock badge" do
    product = create(:product, quantity: 0)
    
    render_inline(described_class.new(product: product))
    
    expect(page).to have_css(".badge", text: "Out of Stock")
    expect(page).to have_css(".product-card--out-of-stock")
  end
  
  it "highlights featured products" do
    render_inline(described_class.new(
      product: product,
      featured: true
    ))
    
    expect(page).to have_css(".product-card--featured")
    expect(page).to have_css(".featured-badge")
  end
  
  it "hides actions when specified" do
    render_inline(described_class.new(
      product: product,
      show_actions: false
    ))
    
    expect(page).not_to have_button("Add to Cart")
  end
end
```

## Organizing Partials

### Directory Structure
```
app/views/
├── layouts/
│   └── application.html.erb
├── shared/
│   ├── _header.html.erb
│   ├── _footer.html.erb
│   ├── _flash_messages.html.erb
│   └── _pagination.html.erb
├── products/
│   ├── index.html.erb
│   ├── show.html.erb
│   ├── _product.html.erb
│   ├── _form.html.erb
│   └── _filters.html.erb
└── components/
    ├── _card.html.erb
    ├── _modal.html.erb
    └── _dropdown.html.erb
```

## Partial Patterns

### Form Partials
```erb
<!-- app/views/products/_form.html.erb -->
<%# locals: (product:, url: nil) %>

<%= form_with model: product, url: url do |form| %>
  <%= render "shared/error_messages", object: product %>
  
  <div class="field">
    <%= form.label :name %>
    <%= form.text_field :name %>
  </div>
  
  <div class="field">
    <%= form.label :description %>
    <%= form.text_area :description %>
  </div>
  
  <%= form.submit %>
<% end %>

<!-- Usage in new.html.erb -->
<%= render "form", product: @product %>

<!-- Usage in edit.html.erb -->
<%= render "form", product: @product %>
```

### Collection Partials with Counters
```erb
<!-- _comment.html.erb -->
<%# locals: (comment:, comment_counter:) %>

<div class="comment" id="comment-<%= comment.id %>">
  <div class="comment-number">#<%= comment_counter + 1 %></div>
  <div class="comment-body">
    <%= comment.body %>
  </div>
</div>

<!-- Usage -->
<%= render partial: "comment", collection: @comments %>
```

## Choosing Between Partials and Components

### Use Partials When:
- Simple HTML structure
- No complex logic
- No need for testing
- Used in single context

### Use View Components When:
- Complex conditional rendering
- Need unit tests
- Reused across multiple views
- Encapsulated business logic
- Performance is critical (components are faster)

## Performance Considerations

### Cache Partials When Appropriate
```erb
<% cache product do %>
  <%= render "product", product: product %>
<% end %>

<!-- With collection caching -->
<%= render partial: "product",
           collection: @products,
           cached: true %>
```

### Avoid Nested Partials
```erb
<!-- BAD - Too many levels -->
<!-- index.html.erb -->
<%= render "products_list" %>
  <!-- _products_list.html.erb -->
  <%= render "product_grid" %>
    <!-- _product_grid.html.erb -->
    <%= render "product_card" %>

<!-- GOOD - Flatter structure or use component -->
<%= render ProductsGridComponent.new(products: @products) %>
```

## Best Practices Summary

1. **Use partials for simple, presentational re-use**
2. **Always pass locals explicitly**
3. **Use strict locals in Rails 7.1+**
4. **Use View Components for complex UI logic**
5. **Test View Components, not partials**
6. **Organize partials logically**
7. **Cache when appropriate**
8. **Avoid deeply nested partials**