# Hard Rules for JavaScript in Rails Applications

## 1. Minimize JavaScript Usage
- Server-rendered HTML is the default
- JavaScript only for interactions impossible on the server
- Every line of JavaScript is a liability

## 2. Embrace Server-Rendered Views
- Use Turbo/Hotwire for most interactions
- Full page reloads are fine
- SPAs are almost never necessary

## 3. No Business Logic in JavaScript
- JavaScript is for UI interactions only
- All business logic stays on the server
- NEVER duplicate validation or calculations

## 4. Use Stimulus for JavaScript Organization
- Stimulus controllers for all JavaScript behavior
- One controller per behavior
- Keep controllers under 100 lines

## 5. No Direct DOM Manipulation
- Use data attributes to connect JavaScript to HTML
- Let Stimulus handle lifecycle
- NEVER use getElementById or querySelector in application code

## 6. Progressive Enhancement Required
- Application MUST work without JavaScript
- JavaScript enhances, never replaces functionality
- Core features work with JavaScript disabled

## 7. No JavaScript in ERB Templates
- NEVER embed JavaScript in views
- Use data attributes to pass data
- Keep JavaScript in separate files

## 8. Third-Party Library Rules
- Minimize external dependencies
- Vendor critical libraries
- Document why each library is necessary
- Regular security audits required

## 9. API Communication
- Use Rails UJS or Turbo for form submissions
- Prefer form submissions over JSON APIs
- If JSON needed, use consistent error handling

## 10. State Management
- Avoid client-side state when possible
- Server is the source of truth
- Use sessionStorage/localStorage sparingly

## 11. Testing Requirements
- Test JavaScript behavior, not implementation
- Use system tests for JavaScript interactions
- Unit test complex Stimulus controllers

## 12. Performance Rules
- Lazy load non-critical JavaScript
- Bundle and minify for production
- Monitor JavaScript bundle size
- Set budget: max 100KB compressed

## 13. Error Handling
- All JavaScript errors must be caught
- Log errors to server monitoring
- Graceful degradation on errors

## 14. Accessibility with JavaScript
- Keyboard navigation for all interactions
- ARIA attributes updated dynamically
- Screen reader testing required

## 15. No Framework Lock-in
- Avoid React/Vue/Angular unless absolutely necessary
- If framework required, isolate to specific pages
- Document why framework is needed

## 16. Security Rules
- NEVER trust user input in JavaScript
- Sanitize all dynamic content
- Use Content Security Policy
- No eval() or innerHTML with user data

## 17. Development Practices
- Use modern JavaScript (ES6+)
- Transpile for browser support
- Source maps in development only
- Lint all JavaScript code

## 18. WebSocket/Real-time Rules
- Use Action Cable for WebSocket needs
- Real-time features only when necessary
- Fallback to polling if WebSocket fails

## 19. Mobile Considerations
- Touch events handled properly
- JavaScript payload minimized for mobile
- Test on real devices, not just responsive mode