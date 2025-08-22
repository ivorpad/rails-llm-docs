# Minimal JavaScript Best Practices

## Core Principle
Embrace server-rendered views by default. JavaScript is a liability - use it only when necessary and leverage the web platform for basic interactions.

## Why JavaScript is a Liability

### You Cannot Control the Runtime
- Different browsers, versions, extensions
- Network conditions vary
- JavaScript can be disabled
- Errors are silent to users

### The Ecosystem is Unstable
- Packages change frequently
- Breaking changes are common
- Security vulnerabilities
- Build tool complexity

## Server-Rendered Views First

### Leverage Turbo for SPA-like Experience
```ruby
# Gemfile
gem "turbo-rails"

# app/views/layouts/application.html.erb
<%= turbo_include_tags %>

# That's it - you get:
# - Fast page transitions
# - Form submissions without full reload
# - Partial page updates
```

### Turbo Frames for Partial Updates
```erb
<!-- Define a frame -->
<%= turbo_frame_tag "product_details" do %>
  <div class="product">
    <h2><%= @product.name %></h2>
    <p><%= @product.description %></p>
    <%= link_to "Edit", edit_product_path(@product) %>
  </div>
<% end %>

<!-- Edit link will only update the frame -->
<!-- edit.html.erb -->
<%= turbo_frame_tag "product_details" do %>
  <%= form_with model: @product do |form| %>
    <%= form.text_field :name %>
    <%= form.text_area :description %>
    <%= form.submit %>
  <% end %>
<% end %>
```

### Turbo Streams for Live Updates
```ruby
# Controller
class CommentsController < ApplicationController
  def create
    @comment = @post.comments.create!(comment_params)
    
    respond_to do |format|
      format.turbo_stream {
        render turbo_stream: turbo_stream.append(
          "comments",
          partial: "comments/comment",
          locals: { comment: @comment }
        )
      }
      format.html { redirect_to @post }
    end
  end
end
```

## Use Web Platform Features

### HTML5 Form Validations
```erb
<!-- No JavaScript needed -->
<%= form_with model: @user do |form| %>
  <!-- Required field -->
  <%= form.email_field :email, required: true %>
  
  <!-- Pattern validation -->
  <%= form.telephone_field :phone, 
                           pattern: "[0-9]{3}-[0-9]{3}-[0-9]{4}",
                           placeholder: "123-456-7890" %>
  
  <!-- Number constraints -->
  <%= form.number_field :age, min: 18, max: 120 %>
  
  <!-- Date constraints -->
  <%= form.date_field :birth_date, 
                      max: Date.today,
                      min: Date.today - 120.years %>
<% end %>
```

### CSS for Interactive Behavior
```css
/* Accordion without JavaScript */
details summary {
  cursor: pointer;
  padding: 10px;
  background: #f0f0f0;
}

details[open] summary {
  background: #e0e0e0;
}

/* Hover menus */
.dropdown {
  position: relative;
}

.dropdown-menu {
  display: none;
  position: absolute;
}

.dropdown:hover .dropdown-menu,
.dropdown:focus-within .dropdown-menu {
  display: block;
}

/* Checkbox hacks for toggles */
.toggle-checkbox {
  display: none;
}

.toggle-content {
  display: none;
}

.toggle-checkbox:checked ~ .toggle-content {
  display: block;
}
```

```erb
<!-- Using CSS-only interactions -->
<details>
  <summary>Show more details</summary>
  <div class="details-content">
    <%= @product.full_description %>
  </div>
</details>

<!-- Toggle without JS -->
<label for="toggle-filters" class="btn">Show Filters</label>
<input type="checkbox" id="toggle-filters" class="toggle-checkbox">
<div class="toggle-content">
  <!-- Filter form here -->
</div>
```

## Custom Elements When Needed

### Creating a Custom Element
```javascript
// app/javascript/elements/auto_submit_element.js
export class AutoSubmitElement extends HTMLElement {
  connectedCallback() {
    this.addEventListener("change", this.#submit)
  }
  
  disconnectedCallback() {
    this.removeEventListener("change", this.#submit)
  }
  
  #submit = (event) => {
    const form = this.closest("form")
    if (form) {
      // Use Turbo for submission
      form.requestSubmit()
    }
  }
}

// Register the element
customElements.define("auto-submit", AutoSubmitElement)
```

```erb
<!-- Usage in view -->
<%= form_with url: products_path, method: :get do |form| %>
  <auto-submit>
    <%= form.select :category, 
                    options_for_select(Category.pluck(:name, :id)),
                    { include_blank: "All Categories" } %>
  </auto-submit>
<% end %>
```

### Stimulus for Reusable Behaviors
```javascript
// app/javascript/controllers/clipboard_controller.js
import { Controller } from "@hotwired/stimulus"

export default class extends Controller {
  static values = { text: String }
  
  copy() {
    navigator.clipboard.writeText(this.textValue)
    
    // Visual feedback
    const originalText = this.element.textContent
    this.element.textContent = "Copied!"
    
    setTimeout(() => {
      this.element.textContent = originalText
    }, 2000)
  }
}
```

```erb
<!-- Usage -->
<button data-controller="clipboard"
        data-clipboard-text-value="<%= @api_key %>"
        data-action="click->clipboard#copy">
  Copy API Key
</button>
```

## Import Maps for Simple JS Management

### Setup Import Maps
```ruby
# Gemfile
gem "importmap-rails"

# config/importmap.rb
pin "application", preload: true
pin "@hotwired/turbo-rails", to: "turbo.min.js", preload: true
pin "@hotwired/stimulus", to: "stimulus.min.js", preload: true

# Pin your custom modules
pin_all_from "app/javascript/elements", under: "elements"
pin_all_from "app/javascript/controllers", under: "controllers"
```

### Application JavaScript
```javascript
// app/javascript/application.js
import "@hotwired/turbo-rails"
import "@hotwired/stimulus"

// Import custom elements
import "elements/auto_submit_element"
import "elements/dropdown_element"

// Import Stimulus controllers
import "controllers"
```

## Progressive Enhancement Pattern

### Start with Working HTML
```erb
<!-- Works without JavaScript -->
<%= form_with url: search_products_path, method: :get do |form| %>
  <%= form.text_field :query, placeholder: "Search..." %>
  <%= form.submit "Search" %>
<% end %>

<!-- Enhance with JavaScript if available -->
<div data-controller="search"
     data-search-url-value="<%= search_products_path %>">
  <input type="text" 
         data-search-target="input"
         data-action="input->search#perform"
         placeholder="Search...">
  
  <div data-search-target="results">
    <!-- Results appear here -->
  </div>
</div>
```

```javascript
// app/javascript/controllers/search_controller.js
import { Controller } from "@hotwired/stimulus"

export default class extends Controller {
  static targets = ["input", "results"]
  static values = { url: String }
  
  perform() {
    clearTimeout(this.timeout)
    
    this.timeout = setTimeout(() => {
      this.#search()
    }, 300)
  }
  
  async #search() {
    const query = this.inputTarget.value
    
    if (query.length < 2) {
      this.resultsTarget.innerHTML = ""
      return
    }
    
    const response = await fetch(`${this.urlValue}?query=${query}`, {
      headers: {
        "Accept": "text/html"
      }
    })
    
    if (response.ok) {
      this.resultsTarget.innerHTML = await response.text()
    }
  }
}
```

## Avoid JavaScript Frameworks

### When You Might Need a Framework
- Building a true SPA (consider if really needed)
- Complex real-time features (but try ActionCable first)
- Heavy client-side computation (but consider server-side)

### Alternatives to Frameworks
```erb
<!-- Instead of React/Vue component -->
<!-- Use ViewComponent or Partial -->
<%= render ProductCardComponent.new(product: @product) %>

<!-- Instead of client-side routing -->
<!-- Use Turbo Drive -->
<%= link_to "Next Page", products_path(page: 2), data: { turbo_frame: "products" } %>

<!-- Instead of state management -->
<!-- Use server state -->
<div data-controller="cart" 
     data-cart-items-value="<%= current_cart.to_json %>">
</div>
```

## Testing JavaScript

### System Tests Catch JS Errors
```ruby
# Ensure JavaScript works in system tests
RSpec.describe "Products", type: :system do
  before do
    driven_by :selenium_chrome_headless
  end
  
  it "filters products dynamically" do
    create(:product, name: "Widget", category: "Hardware")
    create(:product, name: "Gadget", category: "Software")
    
    visit products_path
    
    select "Hardware", from: "category_filter"
    
    # Turbo Frame updates
    within("#products") do
      expect(page).to have_content("Widget")
      expect(page).not_to have_content("Gadget")
    end
  end
  
  it "shows JavaScript errors", :js_errors do
    visit products_path
    
    # Check browser console for errors
    errors = page.driver.browser.logs.get(:browser)
                .select { |e| e.level == "SEVERE" }
    
    expect(errors).to be_empty
  end
end
```

## Best Practices Summary

1. **Default to server-rendered HTML**
2. **Use Turbo for SPA-like features**
3. **Leverage HTML5 and CSS for interactions**
4. **Write custom elements for reusable behaviors**
5. **Use Stimulus for simple interactivity**
6. **Avoid JavaScript frameworks unless necessary**
7. **Always provide non-JS fallbacks**
8. **Test JavaScript in system tests**