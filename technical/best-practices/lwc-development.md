# Lightning Web Components Development

Lightning Web Components should be used as the preferred framework for building UI on the Salesforce platform. This document outlines best practices to ensure maintainable, efficient, and high-quality LWC development.

## Tooling & Setup

- **Formatter:** [Prettier for Salesforce Extensions](https://developer.salesforce.com/docs/platform/sfvscode-extensions/guide/prettier.html)  
  Automatically enforces consistent formatting.

- **Linter:** [`@salesforce/eslint-config-lwc/recommended`](https://developer.salesforce.com/docs/platform/sfvscode-extensions/guide/lwc-linting.html)  
  Run linting before every commit. Integrate it into continuous integration (CI).

- **Type checking**: Use JSDoc for inline type definitions.  
  It improves IDE autocompletion, refactoring accuracy, and developer onboarding.

## Component Structure and Order

Maintain a consistent order of declarations in your component class:

1. **Public properties** (`@api`)
2. **Private properties**
3. **Constructor** (only when needed)
4. **Lifecycle hooks** in execution order:
   - `connectedCallback`
   - `renderedCallback`
   - `disconnectedCallback`
   - `errorCallback`
5. **Public methods** (`@api`)
6. **Wired methods or properties** (`@wire`)
7. **Private/internal methods**

Sort items lexicographically within each section to simplify scanning and code reviews.

## Component Composition Principles

- **Use base components first.**  
  Always prefer standard Lightning base components (`lightning-input`, `lightning-datatable`, `lightning-record-form`, etc.) before building custom ones.

- **Follow SLDS.**  
  Use Salesforce Lightning Design System (SLDS) classes.  
  Avoid custom CSS unless necessary. Extend SLDS, don’t replace it.

- **Favor composition over inheritance.**  
  Encapsulate shared logic in service modules, not abstract base components.  
Composition keeps components small, testable, and reusable.

## Template & Styling Guidelines

- For dynamic CSS classes, use **computed properties**.


## Testing & Validation

- **Unit Tests**: Complex logic (especially helpers) must be covered by Jest tests.
- **Component Tests**: Validate event emissions, reactive updates, and conditional rendering.
- **Linting in CI**: Lint errors should break builds.

## Patterns & Anti-Patterns

### Recommended

- Small, focused components (ideally ≤ 200 lines per file).
- Service modules for logic reuse.
- Defensive async handling (await, error handling).

### Avoid

- Direct DOM manipulation (`this.template.querySelector` should be rare).
- Excessive use of `console.log`; use structured logging.
