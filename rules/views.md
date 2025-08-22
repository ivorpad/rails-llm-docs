# Hard Rules for Rails Views and Templates

## 1. Use Semantic HTML
- ALWAYS use HTML tags for their semantic meaning
- Use `<article>`, `<section>`, `<nav>`, `<header>`, `<footer>` appropriately
- NEVER use `<div>` when a semantic element exists

## 2. One Instance Variable Per View
- Views MUST reference only ONE primary instance variable from the controller
- Additional variables allowed ONLY for reference data (dropdowns, etc.)
- NEVER query the database from views

## 3. No Business Logic in Views
- Views are for presentation ONLY
- Complex conditions belong in helpers or presenters
- NEVER make database queries in views

## 4. ERB Only
- Use ERB for all templates
- AVOID Haml, Slim, or other templating languages
- ERB is maintainable and universally understood

## 5. Partial Rules
- Partials MUST accept only local variables
- NEVER reference instance variables in partials
- Use strict locals (Rails 7.1+) to enforce partial contracts
- Name partials with leading underscore

## 6. Use Rails Helpers for Forms and Links
- ALWAYS use `form_with` for forms
- ALWAYS use `link_to` for links
- ALWAYS use `image_tag` for images
- NEVER write raw HTML for these elements

## 7. No Inline Styles
- NEVER use style attributes
- All styling through CSS classes
- Use data attributes for JavaScript hooks, not classes

## 8. Keep Views Under 50 Lines
- Extract complex sections to partials
- Use view components for reusable UI elements
- Long views are unmaintainable

## 9. Escape User Content
- Rails escapes by default - don't bypass without reason
- NEVER use `raw` or `html_safe` on user input
- Explicitly mark safe content with `sanitize` when needed

## 10. Consistent Naming
- Partial names match their content (e.g., `_widget.html.erb` for widget display)
- View names match controller actions
- Use conventional paths (e.g., `widgets/show.html.erb`)

## 11. No JavaScript in ERB
- NEVER embed JavaScript in ERB templates
- Use data attributes to pass data to JavaScript
- Keep JavaScript in separate files or use Stimulus

## 12. Responsive by Default
- All views MUST work on mobile devices
- Use responsive CSS frameworks or techniques
- Test views at multiple screen sizes

## 13. Accessibility Requirements
- All forms MUST have labels
- All images MUST have alt text
- Use ARIA attributes where appropriate
- Ensure keyboard navigation works

## 14. View Components for Complex UI
- Use ViewComponent gem for reusable, testable UI elements
- Components over 20 lines of markup should be ViewComponents
- Test ViewComponents in isolation