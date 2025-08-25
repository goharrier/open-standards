# Design Patterns

This section documents architectural and design patterns used at Harrier for Salesforce development. These patterns represent proven solutions to recurring problems and provide standardized approaches to common development challenges.

## Patterns Documentation

### [JSON Field Storage Pattern](json-field-storage-pattern)
Leverages LongTextArea fields to store JSON-serialized data for non-transactional, display-oriented information. Provides schema flexibility while avoiding object proliferation for supplementary data that doesn't require querying or workflow processing.

## When to Use Design Patterns

Design patterns should be applied when:
- You encounter recurring problems with established solutions
- You need consistency across the codebase
- The pattern simplifies complex logic
- The benefits outweigh the implementation overhead

## Pattern Selection Criteria

Choose patterns based on:
1. **Problem fit** - Does the pattern address your specific challenge?
2. **Team familiarity** - Can the team maintain pattern-based code?
3. **Performance impact** - Does the pattern meet performance requirements?
4. **Maintenance burden** - Will the pattern simplify long-term maintenance?

## Contributing New Patterns

When documenting new patterns:
1. Base documentation on actual implementations in production code
2. Include concrete examples with proper Apex syntax
3. Explain the problem context and why the pattern matters
4. Provide implementation guidelines and best practices
5. Document common pitfalls and how to avoid them