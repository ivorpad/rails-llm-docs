# Hard Rules for Database and Migrations

## 1. Migration Safety
- NEVER modify migrations after they've been run in production
- Always use strong migrations gem for zero-downtime deployments
- Test rollback for every migration
- Each migration must be reversible

## 2. Schema Integrity
- ALWAYS add database constraints (NOT NULL, foreign keys, unique)
- Database constraints are your last line of defense
- Match model validations with database constraints
- Never rely solely on application-level validation

## 3. Index Requirements
- Index ALL foreign keys
- Index columns used in WHERE clauses
- Index columns used in ORDER BY
- Composite indexes for multi-column queries

## 4. Migration Atomicity
- One concern per migration
- Keep migrations small and focused
- Complex changes split across multiple migrations
- Name migrations descriptively

## 5. Data Migration Rules
- NEVER put data changes in schema migrations
- Create separate rake tasks for data migrations
- Data migrations must be idempotent
- Test data migrations on production-sized datasets

## 6. Column Naming Conventions
- Use snake_case for all columns
- Foreign keys end with `_id`
- Booleans start with `is_` or `has_`
- Timestamps end with `_at`

## 7. Default Values
- Set defaults in database, not application
- NEVER use callbacks for defaults
- Boolean columns MUST have defaults
- Use database functions for computed defaults (e.g., NOW())

## 8. Null Handling
- Avoid NULL when possible
- Use empty strings for optional text
- Use 0 or appropriate number for optional numbers
- Document why NULL is necessary if used

## 9. Table Design
- One table per model
- Avoid STI (Single Table Inheritance)
- Normalize to 3rd normal form minimum
- Denormalize only with measured justification

## 10. Primary Keys
- Always use auto-incrementing integers or UUIDs
- NEVER use composite primary keys
- NEVER use natural keys as primary keys
- Consider UUIDs for distributed systems

## 11. Foreign Key Constraints
- ALWAYS add foreign key constraints
- Define ON DELETE behavior explicitly
- Use CASCADE carefully and document why
- RESTRICT is the safest default

## 12. Performance Considerations
- Keep tables under 1 million rows when possible
- Partition large tables by date or tenant
- Archive old data regularly
- Monitor slow query logs

## 13. Column Types
- Use appropriate types (don't use string for numbers)
- Use ENUM for fixed sets of values
- Use JSON/JSONB sparingly and index appropriately
- Avoid serialized columns

## 14. Migration Testing
- Test migrations on production-like data
- Test both up and down migrations
- Check migration performance on large tables
- Verify indexes are used as expected

## 15. Database Backups
- Automated daily backups minimum
- Test restore procedure regularly
- Keep backups for 30+ days
- Store backups in different location than database

## 16. Query Optimization
- EXPLAIN all complex queries
- Avoid N+1 queries with includes/joins
- Use database views for complex repeated queries
- Monitor query performance in production

## 17. Transaction Usage
- Keep transactions as short as possible
- NEVER make external API calls inside transactions
- Use advisory locks for long-running operations
- Handle deadlocks gracefully

## 18. Database Security
- Use separate users for application and migrations
- NEVER store unencrypted sensitive data
- Use database-level encryption when available
- Audit database access

## 19. Multi-Database Rules
- Avoid cross-database joins
- Keep related data in same database
- Use service objects to coordinate multi-database operations
- Document database boundaries clearly

## 20. Monitoring Requirements
- Monitor disk usage
- Alert on slow queries
- Track connection pool usage
- Monitor replication lag (if applicable)