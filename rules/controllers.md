# Hard Rules for Rails Controllers

## 1. Controllers MUST Only Handle HTTP Concerns
- Parse parameters
- Call service objects or models
- Render responses or redirect
- NEVER put business logic in controllers

## 2. One Instance Variable Per Action
- Each action MUST expose exactly ONE instance variable to the view
- Name it after the resource (e.g., `@widget` for WidgetsController#show)
- Exception: Reference data for forms can be additional variables

## 3. Strong Parameters Are Mandatory
- ALWAYS use strong parameters for mass assignment
- Define private methods for permitted parameters
- NEVER trust user input without whitelisting

## 4. Controller Methods Must Be Under 10 Lines
- If an action exceeds 10 lines, extract logic to service objects
- Complex queries belong in model scopes or query objects
- Multi-step processes belong in service objects

## 5. Use Before Actions Sparingly
- ONLY use for authentication and authorization
- NEVER use for setting instance variables for views
- Avoid complex logic in before_action callbacks

## 6. Standard Action Names Only
- Stick to the seven RESTful actions
- NEVER add custom actions to controllers
- Create new controllers/resources instead of custom actions

## 7. Respond With Appropriate Status Codes
- 200 OK for successful GET requests
- 201 Created for successful POST creating resources
- 204 No Content for successful DELETE
- 422 Unprocessable Entity for validation failures
- NEVER return 200 for errors

## 8. Handle Exceptions at Controller Level
- Use `rescue_from` for handling domain exceptions
- NEVER let exceptions bubble up to Rails defaults in production
- Log errors appropriately before rendering error responses

## 9. Keep Controllers Skinny
- Controllers are HTTP adapters, nothing more
- Business logic belongs in service objects or models
- Complex view logic belongs in helpers or view components

## 10. Inheritance Rules
- Controllers MUST inherit from ApplicationController
- Use controller concerns ONLY for shared HTTP behavior
- NEVER inherit from non-ApplicationController classes

## 11. No Direct Database Access
- Controllers MUST NOT contain SQL queries
- Use model scopes or query objects
- Controllers should not know about database structure

## 12. Flash Messages Consistency
- Use flash[:notice] for success messages
- Use flash[:alert] for error messages
- NEVER use flash for complex data structures