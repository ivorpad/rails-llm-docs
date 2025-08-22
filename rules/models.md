# Hard Rules for Rails Models and Active Records

## 1. Active Records MUST NOT Contain Business Logic
- Active Records are for database access ONLY
- Business logic belongs in service objects or separate domain objects
- Models should only have validations, associations, and scopes

## 2. Keep Models Under 100 Lines
- If a model exceeds 100 lines, it's doing too much
- Extract complex logic to service objects
- Use concerns ONLY for truly reusable database behavior

## 3. Validation Rules
- ALWAYS validate presence of required fields
- Use database constraints AND model validations
- Custom validations should be extracted to validator classes

## 4. Association Rules
- Define all associations explicitly
- Use `dependent:` option to handle cascading deletes
- NEVER use `has_and_belongs_to_many` - use `has_many through:`

## 5. Scope Rules
- Scopes MUST return ActiveRecord::Relation objects
- Keep scopes simple and composable
- Complex queries belong in query objects, not scopes

## 6. Callback Restrictions
- AVOID callbacks whenever possible
- NEVER use callbacks for business logic
- Only use for data normalization (e.g., downcasing emails)
- Prefer service objects to coordinate multi-model operations

## 7. No Direct External API Calls
- Models MUST NOT make HTTP requests
- Models MUST NOT send emails
- Models MUST NOT interact with external services
- These belong in service objects or background jobs

## 8. Attribute Protection
- Use `attr_readonly` for fields that shouldn't change
- NEVER expose internal IDs or sensitive data in `to_json`
- Define explicit serialization methods

## 9. Query Performance Rules
- ALWAYS use `includes` to prevent N+1 queries
- Use `select` to limit columns fetched
- Add database indexes for all foreign keys and frequently queried columns

## 10. Model Naming Conventions
- Models MUST be singular nouns
- Use clear, domain-specific names
- Avoid generic names like "Data" or "Info"

## 11. Default Values
- Set defaults in the database migration
- NEVER set defaults in callbacks
- Use database-level constraints for data integrity

## 12. Model Methods Must Be About the Model
- Methods should operate on the model's own data
- Methods that coordinate multiple models belong in service objects
- Class methods should return collections or perform bulk operations

## 13. Avoid STI (Single Table Inheritance)
- STI adds complexity and performance issues
- Prefer composition over inheritance
- Use separate tables for separate concepts

## 14. Transaction Usage
- Wrap multi-model operations in transactions
- Handle transaction rollbacks explicitly
- NEVER nest transactions without understanding savepoint behavior