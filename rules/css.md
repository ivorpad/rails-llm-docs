# Hard Rules for CSS in Rails Applications

## 1. Adopt a Design System
- MUST have a documented design system before writing CSS
- Define color palettes, spacing scales, typography scales
- NEVER use arbitrary values - everything from the system

## 2. Choose ONE CSS Strategy
- Pick ONE approach: CSS Framework, Object-Oriented CSS, or Functional CSS
- NEVER mix strategies within the same application
- Document the chosen strategy clearly

## 3. No Inline Styles Ever
- NEVER use style attributes in HTML
- NEVER generate inline styles from Rails
- All styling through external CSS classes

## 4. CSS Framework Rules (if using)
- Customize the framework's variables/tokens
- NEVER override framework classes directly
- Create custom utility classes following framework patterns

## 5. Functional CSS Rules (if using Tailwind, etc.)
- Use only the utility classes provided
- Extract common patterns to components
- NEVER write custom CSS alongside functional CSS

## 6. Object-Oriented CSS Rules (if using)
- Follow BEM or chosen naming convention strictly
- One component per file
- Components are independent and reusable

## 7. File Organization
- CSS files mirror the Rails structure
- Component styles in `app/assets/stylesheets/components/`
- Page-specific styles in `app/assets/stylesheets/pages/`
- NEVER put all CSS in one file

## 8. Responsive Design Mandatory
- Mobile-first approach required
- Define breakpoints consistently
- Test at all defined breakpoints

## 9. No Deep Nesting
- Maximum 3 levels of nesting in Sass/SCSS
- Prefer composition over nesting
- Keep specificity low

## 10. Consistent Units
- Use rem for typography
- Use rem or em for spacing
- Use % or viewport units for layout
- NEVER mix units arbitrarily

## 11. Color Management
- Define all colors as variables/custom properties
- NEVER use hex codes directly in components
- Include accessible color contrast ratios

## 12. Living Style Guide Required
- Maintain a living style guide at `/style-guide`
- Document all components and patterns
- Update with every new component addition

## 13. Performance Rules
- Minimize CSS bundle size
- Remove unused CSS before production
- Use CSS modules or scoped styles to prevent bloat

## 14. Accessibility in CSS
- Focus states for all interactive elements
- Sufficient color contrast (WCAG AA minimum)
- Don't rely on color alone for meaning

## 15. No JavaScript-Dependent Styles
- CSS should work without JavaScript
- JavaScript adds behavior, not core styling
- Progressive enhancement over graceful degradation

## 16. Version Control
- Commit CSS changes with their HTML changes
- NEVER commit generated CSS (if using preprocessors)
- Include source maps in development only

## 17. Testing CSS
- Visual regression tests for critical components
- Cross-browser testing required
- Document browser support explicitly