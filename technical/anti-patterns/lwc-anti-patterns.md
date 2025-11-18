# Lightning Web Components Anti-Patterns

## `console.log` statements

Avoid including `console.log` statements in production code.

**Why this matters:**
- Impacts performance.
- User should not see internal logs.

**What to do instead:**
- Remove all `console.log` statements before committing code.
- Use proper logging frameworks for error logging (see [Logging Best Practices](/technical/best-practices/logging.md)).
- Use LWC Debug Mode and browser developer tools for debugging.

## Direct DOM Manipulation

Avoid direct DOM manipulation using standard JavaScript methods (e.g., `document.querySelector`, `element.innerHTML`, etc.).

**Why this matters:**
- Breaks encapsulation provided by LWC.
- Can lead to unpredictable behavior and bugs.

**What to do instead:**  
Use LWC's built-in HTML template directives (e.g. `lwc:ref`).

## Passing a raw value directly to `detail` in `CustomEvent`

```js
this.dispatchEvent(new CustomEvent('testevent', { detail: 'myval' }));
```

**Why this matters:**
When you pass a primitive (string, number, etc.) directly as the detail, you lock yourself into a structure you can’t evolve. If later you need to include more data — e.g. an ID, a type, or some metadata — you’ll have to refactor every consumer of the event. This breaks backward compatibility and makes the event brittle.

**What to do instead:**
Always use an object as the detail payload.

Example (good):

```js
this.dispatchEvent(new CustomEvent('testevent', {
    detail: { value: 'myval' }
}));
```
