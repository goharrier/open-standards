# Service Layer Extraction: Theory and Practice

## Why Service Extraction Matters

Service extraction is about identifying and isolating business logic into dedicated, reusable components. The goal isn't to create services for the sake of it, but to improve maintainability, testability, and clarity of your codebase.

## Understanding What Belongs in a Service

### Core Principle: Business Logic vs Infrastructure Logic

**Business Logic** (belongs in services):
- Calculation rules (discount calculations, pricing, scoring)
- Business workflows (approval processes, state transitions)
- Data transformations (formatting, aggregations, derivations)
- Cross-object operations (updating related records)
- Business validations (complex rules beyond simple field validation)

**Infrastructure Logic** (doesn't belong in services):
- UI rendering and formatting
- Database queries (use Selectors)
- Transaction management (use Unit of Work)
- Field-level validation (use Domain layer)
- Simple CRUD operations without business rules

## The Decision Framework

### When to Extract a Service

Ask yourself these questions:

1. **Is this logic used in multiple places?**
   - If yes → Extract to service
   - If potentially yes in future → Consider extraction
   - If definitely no → Keep it local

2. **Does this logic represent a business capability?**
   - "Calculate customer discount" → Yes, extract
   - "Format date for display" → No, keep in UI layer
   - "Determine eligibility" → Yes, extract

3. **Would a business analyst understand this as a discrete process?**
   - If they'd document it as a business rule → Extract
   - If it's purely technical → Don't extract

4. **Is this logic complex enough to warrant testing in isolation?**
   - Multi-step calculations → Extract
   - Simple field mapping → Don't extract

### When NOT to Extract a Service

- **Premature abstraction**: Don't extract until you see patterns
- **Single-use logic**: If truly used once, keep it where it is
- **Simple transformations**: Basic formatting belongs in the UI
- **Pure data access**: That's what Selectors are for
- **Field validation**: Domain layer handles this

## Identifying Service Boundaries

### The Cohesion Test

A well-defined service should:
- Have a clear, single purpose
- Operate on related data
- Represent one business capability
- Be nameable with a business term

**Good Service Boundaries:**
- `PricingService` - All pricing calculations
- `EligibilityService` - Determination of various eligibilities
- `NotificationService` - All notification logic

**Poor Service Boundaries:**
- `UtilityService` - Grab bag of unrelated functions
- `HelperService` - No clear purpose
- `DataService` - Too broad, mixes concerns

### The Coupling Test

Services should:
- Not depend on UI state
- Not know about trigger context
- Not manage transactions directly
- Accept simple parameters, return simple results

## Practical Extraction Process

### Step 1: Recognize the Smell

Common signs that extraction is needed:
- Copy-pasted business logic across classes
- Controllers with 100+ lines of business logic
- Triggers doing more than coordinating
- Test classes that can't test logic in isolation
- Flow/Process Builder duplicating Apex logic

### Step 2: Map the Logic

Before extracting:
1. List all the business operations
2. Group related operations
3. Identify shared data needs
4. Note external dependencies

Example mapping:
```
Current State: OpportunityController
- Validates opportunity can be closed
- Calculates final discount
- Creates follow-up tasks
- Updates account metrics
- Sends notifications

Proposed Services:
- OpportunityService (closing operations)
- DiscountService (calculations)
- TaskService (task creation)
- NotificationService (alerts)
```

### Step 3: Design the Interface

Think about:
- **Input**: What information does the service need?
- **Output**: What should it return?
- **Side effects**: What else happens?
- **Error cases**: What can go wrong?

Keep interfaces simple:
```apex
// Good: Clear, focused interface
public interface IDiscountService {
    Decimal calculateDiscount(Decimal amount, String customerType);
}

// Poor: Too many responsibilities
public interface IEverythingService {
    Object doStuff(Map<String, Object> params);
}
```

### Step 4: Extract Incrementally

1. **Start with pure functions**: Extract calculations first
2. **Move to stateless operations**: Then extract transformations
3. **Handle stateful operations**: Finally extract complex workflows

Don't try to extract everything at once. Start with the highest-value, lowest-risk extractions.

## Common Patterns and Anti-Patterns

### Pattern: Service Orchestration

When a business process involves multiple steps:
```apex
// OrderService orchestrates multiple services
public void processOrder(Order order) {
    // Validate
    eligibilityService.validateCustomer(order.customerId);
    
    // Calculate
    order.finalPrice = pricingService.calculate(order);
    
    // Execute
    inventoryService.reserve(order.items);
    
    // Notify
    notificationService.sendOrderConfirmation(order);
}
```

### Anti-Pattern: Anemic Services

Services that are just wrappers around data access:
```apex
// Bad: This isn't a service, it's a badly designed selector
public class AccountService {
    public Account getAccount(Id accountId) {
        return [SELECT Id, Name FROM Account WHERE Id = :accountId];
    }
}
```

### Pattern: Parameter Objects

When operations need multiple inputs:
```apex
// Instead of: calculatePrice(amount, discount, tax, shipping, currency)
// Use: calculatePrice(PricingRequest request)
```

### Anti-Pattern: God Service

One service that does everything:
```apex
// Bad: UniversalService with 50+ methods
// Good: Focused services with 5-10 related methods each
```

## Testing Extracted Services

### Key Testing Principles

1. **Test behavior, not implementation**
2. **Use dependency injection for external dependencies**
3. **Test edge cases and error conditions**
4. **Keep tests focused on single service responsibility**

### What Makes Services Testable

- **No direct database access** (inject selectors)
- **No static dependencies** (use dependency injection)
- **Clear inputs and outputs** (avoid side effects)
- **Deterministic behavior** (no hidden state)

## Service Extraction Checklist

Before extracting:
- [ ] Is this truly business logic?
- [ ] Will it be reused?
- [ ] Does it have a clear business purpose?
- [ ] Can it be named with business terms?

During extraction:
- [ ] Define clear interface first
- [ ] Keep services focused
- [ ] Use dependency injection
- [ ] Avoid transaction management in service

After extraction:
- [ ] All callers updated?
- [ ] Tests cover the service?
- [ ] Documentation explains purpose?
- [ ] No logic duplication remains?

## Red Flags to Avoid

1. **Creating services for everything** - Not all logic needs extraction
2. **Services calling services calling services** - Watch your depth
3. **Passing complex objects everywhere** - Keep interfaces simple
4. **Services knowing about UI or database** - Maintain separation
5. **Extraction without refactoring** - Don't just move bad code

## The Business Value Test

Ultimately, ask yourself:
> "If I had to explain this service to a business stakeholder, would they understand its value?"

If yes, you're on the right track. If no, reconsider whether extraction makes sense.

## Conclusion

Service extraction is about finding the right abstraction level for your business logic. It's not about following rules blindly, but about improving code organization in ways that make business sense. Start small, extract incrementally, and always keep the business purpose in mind.