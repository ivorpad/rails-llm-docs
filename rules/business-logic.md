# Hard Rules for Business Logic Organization

## 1. NEVER Put Business Logic in Active Records
- Active Records are for database access ONLY
- Business logic creates high churn in critical classes
- Bugs in Active Records have system-wide impact

## 2. Service Objects for All Business Operations
- Every business operation gets its own service class
- Service objects coordinate models, APIs, and other services
- One service object per business concept

## 3. Service Layer Design Rules
- Service classes in `app/services/`
- Named after what they do (e.g., `WidgetPurchaser`, `UserAuthenticator`)
- Single public method per service (usually `call` or domain-specific name)

## 4. Service Object Size Limits
- Maximum 100 lines per service class
- Single responsibility principle strictly enforced
- Complex operations split into multiple services

## 5. Isolate External Dependencies
- API calls only in service objects or dedicated API clients
- Email sending only through service objects
- File system operations only in service objects

## 6. Transaction Management
- Service objects manage database transactions
- NEVER let controllers manage transactions
- Explicit rollback handling required

## 7. Error Handling Strategy
- Service objects return Result objects (success/failure)
- NEVER throw exceptions for business logic failures
- Controllers handle Result objects appropriately

## 8. Testing Requirements
- Unit test service objects in isolation
- Mock all external dependencies
- Test both success and failure paths
- Service tests should document business rules

## 9. No Framework Dependencies in Core Logic
- Business logic should be framework-agnostic
- Separate Rails dependencies from business rules
- Core domain logic in plain Ruby objects

## 10. Command/Query Separation
- Commands change state (return success/failure)
- Queries return data (never change state)
- NEVER mix commands and queries

## 11. Background Job Integration
- Long-running operations must use background jobs
- Jobs are thin wrappers calling service objects
- NEVER put business logic directly in job classes

## 12. Composition Over Inheritance
- Service objects don't inherit from each other
- Use composition to share behavior
- Mixins only for true cross-cutting concerns

## 13. Clear Naming Conventions
- Service names are verbs or verb phrases
- Method names describe what happens, not how
- Avoid generic names like `process` or `handle`

## 14. Parameter Objects for Complex Inputs
- Use parameter objects when > 3 parameters
- Validate parameters at service boundary
- Don't pass controllers params hash directly

## 15. Domain Events
- Emit domain events for significant business occurrences
- Other services can subscribe to events
- Keeps services loosely coupled

## 16. Service Object Discovery
- If logic doesn't fit in one model, it needs a service
- If operation touches multiple models, use a service
- If operation has business rules beyond validation, use a service

## 17. Avoid Active Record Callbacks
- Callbacks hide business logic
- Service objects make operations explicit
- Easier to test and understand

## 18. Keep Controllers Thin
- Controllers only handle HTTP concerns
- All business logic delegated to services
- Controllers under 20 lines per action

## 19. Documentation Requirements
- Each service must have a class-level comment
- Document business rules and edge cases
- Include examples in comments

## 20. Idempotency
- Critical operations must be idempotent
- Use unique identifiers to prevent double-processing
- Document idempotency guarantees