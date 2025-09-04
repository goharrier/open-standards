# Anti-Patterns

This section documents common anti-patterns observed in Salesforce development and provides guidance on how to recognize and avoid them. Understanding what NOT to do is as important as knowing best practices.

## Anti-Pattern Documentation

This section will contain detailed anti-pattern documentation as patterns are identified and documented based on real-world implementations.

## Why Document Anti-Patterns?

Anti-patterns help teams:

* Recognize problematic code structures early
* Understand the consequences of poor design decisions
* Learn from past mistakes
* Establish code review criteria
* Justify refactoring efforts

## Common Salesforce Anti-Patterns

### 1. **God Class/Object**

Objects or classes that know too much or do too much, violating single responsibility principle.

### 2. **Hard-Coded IDs**

Embedding record IDs, user IDs, or other environment-specific values directly in code or in custom settings / custom metadata.

### 3. **Trigger Recursion**

Uncontrolled recursive trigger execution leading to governor limit violations.

### 4. **SOQL in Loops**

Executing queries inside loops, quickly hitting governor limits.

### 5. **Missing Bulkification**

Code that only works for single records, failing when processing bulk data.

### 6. **Excessive Field Count**

Objects with hundreds of fields, indicating poor data model design.

### 7. **Process Builder Proliferation**

Multiple Process Builders on the same object causing order dependency issues.

## Identifying Anti-Patterns

Look for these warning signs:

* Difficult to understand code
* Frequent production issues
* High maintenance burden
* Poor performance
* Inability to extend functionality
* Excessive technical debt

## Remediation Strategies

When anti-patterns are identified:

1. Document the issue and its impact
2. Assess the risk of leaving it versus fixing it
3. Plan incremental refactoring
4. Establish tests before refactoring
5. Implement the improved pattern
6. Monitor for regression

## Prevention

Prevent anti-patterns through:

* Code reviews with pattern checklists
* Architectural standards documentation
* Developer training and mentoring
* Static code analysis tools
* Regular technical debt assessments
