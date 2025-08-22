# Hard Rules for Rails Helpers

## 1. Helpers Are for View Logic Only
- Helpers generate HTML or format data for display
- NEVER put business logic in helpers
- NEVER make database queries in helpers

## 2. Don't Conflate Helpers with Your Domain
- Helpers are about presentation, not business rules
- Domain logic belongs in models or service objects
- Keep the separation clear and absolute

## 3. Helper Method Size Limit
- Helper methods MUST be under 10 lines
- Complex helpers should be extracted to view components
- Single responsibility per helper method

## 4. Use Rails Built-in Helpers
- ALWAYS use Rails helpers for common tasks (link_to, form_with, etc.)
- Don't reinvent what Rails provides
- Extend Rails helpers rather than replacing them

## 5. Helper Organization
- Configure `config.action_controller.include_all_helpers = false`
- Keep helpers modular and specific to their controllers
- ApplicationHelper for truly global helpers only

## 6. Testing Requirements
- ALL helpers MUST have tests
- Test both success cases and edge cases
- Helpers should be pure functions when possible

## 7. Naming Conventions
- Helper method names must clearly indicate their purpose
- Use prefixes for grouped functionality (e.g., `format_`, `render_`)
- Avoid generic names like `process` or `handle`

## 8. No State in Helpers
- Helpers MUST NOT maintain state between calls
- Helpers are functional, not object-oriented
- Pass all required data as parameters

## 9. HTML Safety
- Use `tag` and `content_tag` helpers for generating HTML
- NEVER concatenate strings to build HTML
- Always escape user input unless explicitly safe

## 10. Prefer View Components for Complex Logic
- If a helper needs multiple private methods, use a view component
- If a helper maintains internal state, use a view component
- If a helper exceeds 20 lines total, use a view component

## 11. Helper Parameters
- Limit helper methods to 3 parameters maximum
- Use options hash for multiple optional parameters
- Document all parameters clearly

## 12. Global UI State Only
- Helpers can expose global UI state (current user, feature flags)
- Helpers can format common data types (dates, currency)
- Everything else should be passed as parameters

## 13. No Business Rule Validation
- Helpers display data, they don't validate it
- Business rules belong in models or service objects
- Helpers should work with already-validated data