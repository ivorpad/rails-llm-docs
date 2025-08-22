# Helpers Usage Best Practices

## Core Principle
Helpers are best for exposing global UI state and generating markup. Don't conflate helpers with your domain logic.

## What Helpers Are Good For

### Global UI State and Logic
```ruby
# app/helpers/application_helper.rb
module ApplicationHelper
  # Current page detection
  def current_page_class(path)
    "active" if current_page?(path)
  end
  
  # User state helpers
  def user_signed_in?
    current_user.present?
  end
  
  def admin_user?
    current_user&.admin?
  end
  
  # Environment helpers
  def development_environment?
    Rails.env.development?
  end
  
  def show_debug_info?
    development_environment? && params[:debug].present?
  end
end
```

### Small, Inline UI Components
```ruby
module ApplicationHelper
  # Status badges
  def status_badge(status)
    css_class = case status
                when "active" then "badge-success"
                when "pending" then "badge-warning"
                when "inactive" then "badge-danger"
                else "badge-secondary"
                end
    
    content_tag(:span, status.humanize, class: "badge #{css_class}")
  end
  
  # Flash message helpers
  def flash_class(level)
    case level.to_sym
    when :notice then "alert-info"
    when :success then "alert-success"
    when :error, :alert then "alert-danger"
    when :warning then "alert-warning"
    else "alert-info"
    end
  end
  
  # Icon helpers
  def icon(name, text = nil, options = {})
    content_tag(:span, class: "icon-wrapper #{options[:class]}") do
      concat content_tag(:i, "", class: "icon icon-#{name}")
      concat content_tag(:span, text) if text.present?
    end
  end
end
```

## What NOT to Put in Helpers

### BAD: Domain Logic in Helpers
```ruby
# BAD - Business logic doesn't belong in helpers
module ProductsHelper
  def calculate_discount(product, user)
    base_discount = product.category.discount_rate
    if user.premium?
      base_discount + 0.1
    elsif user.member_since < 1.year.ago
      base_discount + 0.05
    else
      base_discount
    end
  end
  
  def can_purchase?(product, user)
    product.in_stock? && 
      user.verified? && 
      !user.blocked? &&
      product.available_in_region?(user.region)
  end
end

# GOOD - Keep business logic in models/services
class ProductPricing
  def discount_for(product:, user:)
    # Discount calculation logic
  end
end

class ProductAvailability
  def can_purchase?(product:, user:)
    # Purchase eligibility logic
  end
end
```

## Markup Generation Helpers

### Use Rails APIs
```ruby
module ApplicationHelper
  # Building forms
  def error_messages_for(object)
    return if object.errors.empty?
    
    content_tag(:div, class: "error-messages alert alert-danger") do
      content_tag(:ul) do
        object.errors.full_messages.map do |msg|
          content_tag(:li, msg)
        end.join.html_safe
      end
    end
  end
  
  # Links with icons
  def link_with_icon(text, path, icon_name, options = {})
    link_to path, options do
      concat content_tag(:i, "", class: "icon icon-#{icon_name}")
      concat " "
      concat text
    end
  end
  
  # Navigation helpers
  def nav_link(text, path, options = {})
    css_class = current_page?(path) ? "nav-link active" : "nav-link"
    options[:class] = [css_class, options[:class]].compact.join(" ")
    
    link_to text, path, options
  end
end
```

### Complex Markup Generation
```ruby
module TableHelper
  def data_table(records, columns, options = {})
    content_tag(:table, class: "table #{options[:class]}") do
      concat table_header(columns)
      concat table_body(records, columns)
    end
  end
  
  private
  
  def table_header(columns)
    content_tag(:thead) do
      content_tag(:tr) do
        columns.map { |col| content_tag(:th, col[:label]) }.join.html_safe
      end
    end
  end
  
  def table_body(records, columns)
    content_tag(:tbody) do
      records.map do |record|
        content_tag(:tr) do
          columns.map do |col|
            value = col[:value].is_a?(Proc) ? col[:value].call(record) : record.send(col[:value])
            content_tag(:td, value)
          end.join.html_safe
        end
      end.join.html_safe
    end
  end
end

# Usage
<%= data_table @users, [
  { label: "Name", value: :name },
  { label: "Email", value: :email },
  { label: "Status", value: ->(u) { status_badge(u.status) } }
] %>
```

## Configuration Strategies

### Module-based Helpers
```ruby
# config/application.rb
# Option 1: Include all helpers everywhere (default)
config.action_controller.include_all_helpers = true

# Option 2: Only include controller-specific helpers
config.action_controller.include_all_helpers = false

# With Option 2, organize helpers by controller
# app/helpers/products_helper.rb
module ProductsHelper
  def product_image(product, size: :thumb)
    if product.image.attached?
      image_tag product.image.variant(resize_to_limit: size_dimensions(size))
    else
      image_tag "product-placeholder.png", alt: "No image"
    end
  end
  
  private
  
  def size_dimensions(size)
    case size
    when :thumb then [100, 100]
    when :medium then [300, 300]
    when :large then [600, 600]
    end
  end
end
```

### Consolidating Helpers
```ruby
# If you prefer one file for small apps
# app/helpers/application_helper.rb
module ApplicationHelper
  # Navigation
  def nav_link(text, path)
    # ...
  end
  
  # Forms
  def error_messages_for(object)
    # ...
  end
  
  # Formatting
  def format_currency(amount)
    number_to_currency(amount)
  end
  
  def format_date(date)
    date.strftime("%B %d, %Y") if date.present?
  end
end
```

## Helper Method Pattern

### Share Logic Between Controllers and Views
```ruby
class ApplicationController < ActionController::Base
  helper_method :current_user, :signed_in?, :current_cart
  
  private
  
  def current_user
    @current_user ||= User.find(session[:user_id]) if session[:user_id]
  end
  
  def signed_in?
    current_user.present?
  end
  
  def current_cart
    @current_cart ||= Cart.find(session[:cart_id]) || Cart.create
  end
end

# Now available in both controllers and views
# In controller:
def show
  redirect_to login_path unless signed_in?
end

# In view:
<% if signed_in? %>
  Welcome, <%= current_user.name %>
<% end %>
```

## Testing Helpers

### Test Helpers in Isolation
```ruby
# spec/helpers/application_helper_spec.rb
RSpec.describe ApplicationHelper do
  describe "#status_badge" do
    it "returns success badge for active status" do
      result = helper.status_badge("active")
      expect(result).to include("badge-success")
      expect(result).to include("Active")
    end
    
    it "returns warning badge for pending status" do
      result = helper.status_badge("pending")
      expect(result).to include("badge-warning")
    end
  end
  
  describe "#nav_link" do
    it "adds active class to current page" do
      allow(helper).to receive(:current_page?).and_return(true)
      
      result = helper.nav_link("Home", "/")
      expect(result).to include("active")
    end
  end
end
```

### Test Complex Markup
```ruby
RSpec.describe TableHelper do
  describe "#data_table" do
    let(:users) do
      [
        double(name: "Alice", email: "alice@example.com"),
        double(name: "Bob", email: "bob@example.com")
      ]
    end
    
    let(:columns) do
      [
        { label: "Name", value: :name },
        { label: "Email", value: :email }
      ]
    end
    
    it "generates table with headers and data" do
      result = helper.data_table(users, columns)
      
      expect(result).to have_css("table")
      expect(result).to have_css("thead th", text: "Name")
      expect(result).to have_css("tbody td", text: "Alice")
      expect(result).to have_css("tbody td", text: "alice@example.com")
    end
  end
end
```

## Format Helpers

### Date and Time Formatting
```ruby
module DateHelper
  def relative_time(datetime)
    return "Never" if datetime.nil?
    
    if datetime > Time.current
      "in #{time_ago_in_words(datetime)}"
    else
      "#{time_ago_in_words(datetime)} ago"
    end
  end
  
  def date_range(start_date, end_date)
    if start_date.to_date == end_date.to_date
      start_date.strftime("%B %d, %Y")
    elsif start_date.year == end_date.year
      if start_date.month == end_date.month
        "#{start_date.strftime('%B %d')} - #{end_date.day}, #{end_date.year}"
      else
        "#{start_date.strftime('%B %d')} - #{end_date.strftime('%B %d, %Y')}"
      end
    else
      "#{start_date.strftime('%B %d, %Y')} - #{end_date.strftime('%B %d, %Y')}"
    end
  end
end
```

### Text Formatting
```ruby
module TextHelper
  def markdown_to_html(text)
    return "" if text.blank?
    
    renderer = Redcarpet::Render::HTML.new(
      filter_html: true,
      no_images: false,
      no_links: false,
      safe_links_only: true
    )
    
    markdown = Redcarpet::Markdown.new(renderer)
    markdown.render(text).html_safe
  end
  
  def highlight_search_terms(text, terms)
    return text if terms.blank?
    
    highlight(text, terms.split, highlighter: '<mark>\1</mark>')
  end
end
```

## Best Practices Summary

1. **Use helpers for UI state and markup generation**
2. **Keep domain logic out of helpers**
3. **Use Rails' built-in tag helpers**
4. **Test complex helpers**
5. **Consider View Components for complex UI**
6. **Use helper_method to share between controllers and views**
7. **Organize helpers logically**
8. **Keep helpers simple and focused**