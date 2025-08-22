# Hard Rules for Rails Routing

## 1. ALWAYS Use Canonical Routes that Conform to Rails' Defaults
- Routes MUST follow the RESTful convention: index, show, new, create, edit, update, destroy
- Use `resources` helper for all standard resources
- NEVER deviate from Rails naming conventions without exceptional justification

## 2. NEVER Configure Routes That Aren't Being Used
- Remove ALL unused routes immediately
- Use `only:` or `except:` to limit generated routes
- Dead routes are security vulnerabilities and maintenance debt

## 3. Vanity URLs MUST Redirect to Canonical Routes
- NEVER implement business logic in vanity URL controllers
- Always use 301/302 redirects to the canonical resource URL
- Keep vanity URLs separate from core resource logic

## 4. NEVER Create Custom Actions - Create More Resources Instead
- Replace custom actions with new resources
- Bad: `POST /widgets/:id/rate`
- Good: `POST /widget_ratings` with widget_id parameter
- Every custom action is a sign of poor resource modeling

## 5. Use Nested Routes Strategically and Sparingly
- ONLY nest routes when there's a true hierarchical relationship
- NEVER nest more than one level deep
- Prefer flat routes with query parameters over deep nesting

## 6. Namespacing Rules
- Use namespacing ONLY for distinct application areas (admin, API versions)
- NEVER use namespacing as a substitute for proper resource design
- Namespaced routes should represent entirely different contexts

## 7. Route Order and Priority
- Place more specific routes BEFORE generic ones
- Root route MUST be defined first
- Catch-all routes (if any) MUST be last

## 8. Parameter Constraints
- ALWAYS use constraints for non-standard route parameters
- Validate ID formats at the route level when possible
- Use route constraints to prevent invalid requests from reaching controllers

## 9. HTTP Verb Usage
- GET requests MUST be idempotent and safe
- NEVER use GET for state-changing operations
- Use appropriate verbs: POST for creation, PATCH/PUT for updates, DELETE for removal

## 10. Route Documentation
- Every non-standard route MUST have a comment explaining its purpose
- Document any deviations from RESTful conventions
- Keep routes.rb as the single source of truth for URL structure