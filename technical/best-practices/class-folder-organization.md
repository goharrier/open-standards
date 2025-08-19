# Class Folder Organization Patterns

## Introduction

Salesforce's default structure puts all Apex classes in a single `classes` folder, which becomes unmanageable at scale. This document presents patterns for organizing classes into logical subfolders, based on production implementations with hundreds of classes per package.

## Why Folder Organization Matters

### The Problem with Flat Structure

```
// ❌ Default Salesforce structure with 200+ files
classes/
├── AccountService.cls
├── AccountServiceImpl.cls
├── AccountServiceTest.cls
├── AccountSelector.cls
├── AccountSelectorTest.cls
├── AccountDomain.cls
├── AccountDomainTest.cls
├── AccountTriggerHandler.cls
├── AccountFactory.cls
├── IAccountService.cls
├── IAccountSelector.cls
├── OrderService.cls
├── OrderServiceImpl.cls
├── OrderServiceTest.cls
... (185 more files)
```

**Problems:**
- Impossible to find related classes
- No clear architecture visible
- Merge conflicts on every PR
- IDE performance issues
- New developers lost

## Folder Organization Patterns

### 1. Layer-Based Organization (fflib Pattern)

Organize by architectural layer following enterprise patterns:

```
classes/
├── Application.cls                    # Dependency injection root
├── constants/                         # System-wide constants
│   ├── CaseStatuses.cls
│   ├── DocumentTypes.cls
│   └── ProfileNames.cls
├── domains/                          # Domain layer (business logic)
│   ├── Accounts.cls
│   ├── Cases.cls
│   ├── Orders.cls
│   └── interfaces/
│       ├── IAccounts.cls
│       ├── ICases.cls
│       └── IOrders.cls
├── selectors/                        # Data access layer
│   ├── AccountsSelector.cls
│   ├── CasesSelector.cls
│   ├── OrdersSelector.cls
│   └── interfaces/
│       ├── IAccountsSelector.cls
│       ├── ICasesSelector.cls
│       └── IOrdersSelector.cls
├── services/                         # Service layer (orchestration)
│   ├── implementations/
│   │   ├── AccountServiceImpl.cls
│   │   ├── OrderServiceImpl.cls
│   │   └── PricingServiceImpl.cls
│   └── interfaces/
│       ├── IAccountService.cls
│       ├── IOrderService.cls
│       └── IPricingService.cls
├── controllers/                      # UI controllers (when needed)
├── factories/                        # Object factories
│   ├── AccountFactory.cls
│   └── OrderFactory.cls
├── triggerHandlers/                  # Trigger logic
│   ├── AccountsTriggerHandler.cls
│   └── OrdersTriggerHandler.cls
└── flowActions/                      # Flow/Process Builder actions
    ├── CaseStatusChecker.cls
    └── OrderValidator.cls
```

### 2. Test Organization Pattern

Tests mirror the source structure in a separate `test` folder:

```
src/
├── main/
│   └── default/
│       └── classes/
│           ├── domains/
│           │   └── Accounts.cls
│           ├── selectors/
│           │   └── AccountsSelector.cls
│           └── services/
│               └── implementations/
│                   └── AccountServiceImpl.cls
└── test/
    └── default/
        └── classes/
            ├── TestDataFactory.cls           # Shared test utilities
            ├── domains/
            │   └── AccountsTest.cls          # Mirrors main structure
            ├── selectors/
            │   └── AccountsSelectorTest.cls
            └── services/
                └── AccountServiceTest.cls
```

### 3. Sub-Package Organization

For sub-packages within a larger package (modular within modular):

```
classes/
├── Application.cls
├── customer/                         # Customer feature area
│   ├── controllers/
│   │   └── CustomerPortalController.cls
│   ├── domains/
│   │   └── Customers.cls
│   ├── selectors/
│   │   └── CustomersSelector.cls
│   └── services/
│       └── CustomerService.cls
├── order/                           # Order feature area
│   ├── controllers/
│   │   └── OrderWizardController.cls
│   ├── domains/
│   │   └── Orders.cls
│   ├── selectors/
│   │   └── OrdersSelector.cls
│   └── services/
│       └── OrderService.cls
└── shared/                          # Shared across features
    ├── utils/
    │   └── StringUtils.cls
    └── exceptions/
        └── BusinessException.cls
```

### 4. Interface Segregation Pattern

Interfaces in dedicated subfolders for clear contracts:

```
classes/
├── services/
│   ├── interfaces/              # Service contracts
│   │   ├── ICustomerService.cls
│   │   ├── IOrderService.cls
│   │   └── IPricingService.cls
│   └── implementations/         # Concrete implementations
│       ├── CustomerServiceImpl.cls
│       ├── OrderServiceImpl.cls
│       └── PricingServiceImpl.cls
├── domains/
│   ├── interfaces/              # Domain contracts
│   │   ├── ICustomers.cls
│   │   └── IOrders.cls
│   └── Customers.cls            # Concrete domains
└── selectors/
    ├── interfaces/              # Selector contracts
    │   ├── ICustomersSelector.cls
    │   └── IOrdersSelector.cls
    └── CustomersSelector.cls    # Concrete selectors
```

## Package-Specific Application Classes

### The Multi-Application Pattern

In modular architectures, each package has its own `Application_{XX}` class with package-specific suffix:

```apex
// Core package
public class Application {
    // Core selectors for standard objects
    public static final fflib_Application.SelectorFactory Selector =
        new fflib_Application.SelectorFactory(
            new Map<SObjectType, Type>{
                Case.SObjectType => CasesSelector.class,
                Account.SObjectType => AccountsSelector.class
            });
}

// Damage Assessment package
public class Application_DA {
    // Package-specific selectors
    public static final fflib_Application.SelectorFactory Selector =
        new fflib_Application.SelectorFactory(
            new Map<SObjectType, Type>{
                Case.SObjectType => CasesSelector_DA.class,  // Different selector!
                DamageAssessmentAnswer__c.SObjectType => DamageAssessmentAnswersSelector_DA.class,
                Damages__c.SObjectType => DamagesSelector_DA.class
            });
}

// Escalation Management package  
public class Application_EM {
    // Another package-specific selector for Case
    public static final fflib_Application.SelectorFactory Selector =
        new fflib_Application.SelectorFactory(
            new Map<SObjectType, Type>{
                Case.SObjectType => CasesSelector_EM.class  // Yet another selector!
            });
}
```

### Why Multiple Selectors for Same Object?

Each package needs different fields from the same object:

```apex
// Core CasesSelector - Basic fields
public class CasesSelector extends fflib_SObjectSelector {
    public List<Schema.SObjectField> getSObjectFieldList() {
        return new List<Schema.SObjectField>{
            Case.Id,
            Case.CaseNumber,
            Case.Status,
            Case.Subject
        };
    }
}

// Escalation Management CasesSelector_EM - Escalation fields
public class CasesSelector_EM extends fflib_SObjectSelector {
    public List<Schema.SObjectField> getSObjectFieldList() {
        return new List<Schema.SObjectField>{
            Case.Id,
            Case.CaseNumber,
            Case.EscalationReason__c,
            Case.EscalationDetails__c,
            Case.SubStatus__c
        };
    }
}

// Damage Assessment CasesSelector_DA - Property fields
public class CasesSelector_DA extends fflib_SObjectSelector {
    public List<Schema.SObjectField> getSObjectFieldList() {
        return new List<Schema.SObjectField>{
            Case.Id,
            Case.CaseNumber,
            Case.PropertyDamageType__c,
            Case.EstimatedDamageAmount__c
        };
    }
}
```

### Package Suffix Convention

Use consistent 2-3 letter suffixes across all layers:

```
Package: damage-assessment
Suffix: _DA

Classes:
- Application_DA
- CasesSelector_DA
- DamagesSelector_DA
- DamageService_DA
- DamageController_DA
```

### Selector Factory Pattern

Each package-specific selector implements the factory pattern:

```apex
public class CasesSelector_EM extends fflib_SObjectSelector 
    implements ICasesSelector_EM {
    
    // Factory method for user mode
    public static ICasesSelector_EM newInstance() {
        ICasesSelector_EM selector = 
            (ICasesSelector_EM) implementationType.newInstance();
        selector.setDataAccess(fflib_SObjectSelector.DataAccess.USER_MODE);
        return selector;
    }
    
    // Factory method for system mode
    public static ICasesSelector_EM newElevatedInstance() {
        ICasesSelector_EM selector = 
            (ICasesSelector_EM) implementationType.newInstance();
        selector.setDataAccess(fflib_SObjectSelector.DataAccess.SYSTEM_MODE);
        return selector;
    }
    
    // Package-specific queries
    public List<Case> selectByIdsWithEscalationDetails(Set<Id> ids) {
        return Database.query(
            newQueryFactory()
                .selectField('Owner.Name')
                .selectField('EscalationTeam__r.Name')
                .setCondition('Id IN :ids')
                .toSOQL()
        );
    }
}
```

### Using Package-Specific Components

Services use the appropriate Application class:

```apex
// In Escalation Management service
public class EscalationService_EM {
    
    public void processEscalations(Set<Id> caseIds) {
        // Use package-specific selector
        ICasesSelector_EM selector = CasesSelector_EM.newInstance();
        List<Case> cases = selector.selectByIdsWithEscalationDetails(caseIds);
        
        // Use package-specific UoW
        fflib_ISObjectUnitOfWork uow = Application_EM.UnitOfWork.newInstance();
        
        // Process with package-specific logic
        processEscalationLogic(cases, uow);
        
        uow.commitWork();
    }
}
```

## Naming Conventions

### Folder Names
- **Lowercase**: `services`, `domains`, `selectors`
- **Plural for collections**: `controllers`, `factories`
- **Descriptive**: `implementations` not `impl`

### Class Names
- **Services**: `{Entity}Service` + `I{Entity}Service` interface
- **Selectors**: `{Entity}sSelector` + `I{Entity}sSelector` interface
- **Domains**: `{Entity}s` (plural) + `I{Entity}s` interface
- **Controllers**: `{Feature}Controller`
- **Tests**: `{ClassName}Test`

### File Placement Rules

```apex
// Service Interface
// Path: services/interfaces/IAccountService.cls
public interface IAccountService {
    Account createAccount(AccountRequest request);
}

// Service Implementation
// Path: services/implementations/AccountServiceImpl.cls
public class AccountServiceImpl implements IAccountService {
    public Account createAccount(AccountRequest request) {
        // Implementation
    }
}

// Service Facade (optional)
// Path: services/AccountService.cls
public class AccountService {
    private static IAccountService service() {
        return (IAccountService) Application.Service.newInstance(IAccountService.class);
    }
    
    public static Account createAccount(AccountRequest request) {
        return service().createAccount(request);
    }
}

// Test
// Path: ../test/default/classes/services/AccountServiceTest.cls
@IsTest
private class AccountServiceTest {
    // Test implementation
}
```

## Benefits of Folder Organization

### 1. Improved Developer Experience
- **Find files faster**: Related classes grouped together
- **Understand architecture**: Structure visible in folder layout
- **Reduce cognitive load**: Clear separation of concerns

### 2. Better Code Quality
- **Enforce architecture**: Folder structure enforces patterns
- **Reduce coupling**: Clear boundaries between layers
- **Easier reviews**: Reviewers know where to look

### 3. Team Scalability
- **Parallel development**: Teams work in different folders
- **Clear ownership**: Folders can have code owners
- **Onboarding**: New developers understand structure

### 4. Maintenance Benefits
- **Easier refactoring**: Related code in one place
- **Dependency tracking**: Clear layer dependencies
- **Test organization**: Tests mirror source structure

## Anti-Patterns to Avoid

### 1. Over-Nesting
```
// ❌ Too many levels
classes/
└── services/
    └── customer/
        └── implementations/
            └── v2/
                └── async/
                    └── CustomerServiceAsyncV2Impl.cls
```

### 2. Inconsistent Organization
```
// ❌ Mixed patterns
classes/
├── services/          # Layer-based
├── customer/          # Feature-based
├── utils/            # Type-based
└── AccountService.cls # Flat!
```

### 3. Breaking Salesforce Conventions
```
// ❌ Don't rename required folders
src/
└── apex_classes/  # Must be 'classes'
```

## Migration Strategy

### Phase 1: Plan Structure
1. Audit existing classes
2. Map to target folders
3. Identify dependencies
4. Create folder structure

### Phase 2: Gradual Migration
```bash
# Move layer by layer
1. Constants first (no dependencies)
2. Interfaces (contracts)
3. Selectors (data layer)
4. Domains (business logic)
5. Services (orchestration)
6. Controllers (UI layer)
7. Tests (mirror structure)
```

### Phase 3: Update References
```apex
// Before: Flat structure
AccountService service = new AccountService();

// After: With Application pattern
IAccountService service = (IAccountService) 
    Application.Service.newInstance(IAccountService.class);
```

## Production Implementation Results

Effective folder organization in production systems shows:

```
src/{package}/
├── main/default/classes/
│   ├── Application.cls           # Root application class
│   ├── constants/                # System-wide constants
│   ├── domains/                  # Domain logic and business rules
│   │   └── interfaces/          # Domain contracts
│   ├── selectors/               # Data access layer
│   │   └── interfaces/          # Selector contracts
│   ├── services/                # Business orchestration
│   │   ├── implementations/    # Concrete implementations
│   │   └── interfaces/         # Service contracts
│   ├── factories/              # Object construction patterns
│   ├── flowActions/            # Declarative tool integrations
│   └── triggerHandlers/        # Trigger logic separation
└── test/default/classes/
    ├── TestDataFactory.cls     # Shared test utilities
    ├── domains/                # Tests mirror main structure
    ├── selectors/             # Tests mirror main structure
    ├── services/              # Tests mirror main structure
    ├── flowActions/           # Tests mirror main structure
    └── triggerHandlers/       # Tests mirror main structure
```

**Why this works:**
- **Predictable navigation**: Developers know exactly where to find code
- **Clear architecture**: Folder structure enforces architectural patterns
- **Reduced conflicts**: Teams work in separate folders without collision
- **Scalable growth**: Structure maintains clarity as the codebase grows
- **Test organization**: Tests automatically follow the same structure

## Conclusion

Organizing classes into folders is essential for:
- **Maintainability** at scale
- **Architectural clarity**
- **Team productivity**
- **Code quality**

The key is choosing a consistent pattern that fits your architecture and sticking to it across all packages.