# Hard Rules for Testing Rails Applications

## 1. Test Coverage Requirements
- Minimum 80% code coverage
- 100% coverage for service objects
- 100% coverage for business logic
- Every bug fix requires a regression test

## 2. Test Pyramid Structure
- Many unit tests (fast, isolated)
- Some integration tests (component interaction)
- Few system tests (end-to-end with browser)
- Ratio approximately 70:20:10

## 3. Unit Test Requirements
- Test models, services, and components in isolation
- Mock all external dependencies
- Each test tests ONE thing
- Tests must run in under 0.1 seconds each

## 4. Service Object Testing
- Test all success paths
- Test all failure paths
- Test edge cases and boundary conditions
- Mock all database calls and external services

## 5. Controller Testing
- Test HTTP responses and status codes
- Test parameter filtering
- Test authentication and authorization
- DO NOT test business logic in controller tests

## 6. Model Testing
- Test all validations
- Test all scopes
- Test associations configuration
- Test any model methods

## 7. System Test Guidelines
- Test critical user paths only
- Use system tests for JavaScript interactions
- Keep system tests under 10 seconds each
- Run system tests in CI only

## 8. Test Naming Conventions
- Describe what is being tested
- Use "should" or "must" in test names
- Group related tests in contexts/describes
- Test file names match source file names

## 9. Factory/Fixture Rules
- Use factories (FactoryBot) over fixtures
- Keep factories minimal and valid
- Don't create associated records unless needed
- Use traits for variations

## 10. Test Data Management
- Each test must be independent
- Clean database between tests
- Never rely on test execution order
- Use database transactions for speed

## 11. Mocking and Stubbing
- Mock external services always
- Stub time-dependent code
- Don't mock what you own
- Verify mocks are called correctly

## 12. Test Performance
- Full test suite runs in under 5 minutes
- Unit tests run in under 30 seconds
- Parallelize tests in CI
- Profile and optimize slow tests

## 13. Continuous Integration
- All tests must pass before merge
- No skipped or pending tests in main branch
- Run tests on every push
- Automated deployment only after tests pass

## 14. Test Documentation
- Tests serve as documentation
- Write clear test descriptions
- Include context for complex scenarios
- Add comments for non-obvious assertions

## 15. View Testing
- Test ViewComponents in isolation
- Test complex helpers with unit tests
- Use system tests for user interactions
- Don't test simple ERB templates

## 16. API Testing
- Test all endpoints
- Test authentication and authorization
- Test error responses
- Test pagination and filtering

## 17. Background Job Testing
- Test job queuing
- Test job execution
- Test job failures and retries
- Mock actual job performance in integration tests

## 18. Security Testing
- Test authorization on all controllers
- Test SQL injection prevention
- Test XSS prevention
- Test CSRF protection

## 19. Test Refactoring
- Keep test code DRY
- Extract common setup to methods
- Use shared examples for common behavior
- Maintain test code like production code

## 20. No Test Pollution
- Tests don't write to filesystem
- Tests don't make real API calls
- Tests don't send real emails
- Tests clean up after themselves