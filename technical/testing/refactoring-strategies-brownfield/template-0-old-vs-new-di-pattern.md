# Apex DI Pattern Migration Guide

**Old Pattern** (Application Factory + Custom Metadata) **-> New Pattern** (replaceWith + Self-Contained DI)

*Reference document for developers migrating selectors, services, and unit of work classes.*

---

## Summary of Changes

| Aspect | **OLD** Application Factory | **NEW** replaceWith |
|---|---|---|
| **Central registry** | `Application_*.cls` class with factory maps | Not needed -- each class manages its own DI |
| **Custom Metadata** | Required for binding configuration | **No longer needed** -- bindings are in code |
| **Parent class (Selectors)** | `fflib_SObjectSelector` | `fflib_SObjectSelector2` |
| **newInstance()** | Delegates to `Application_*.Selector.newInstance(SObjectType)` | Self-contained -- checks local static fields |
| **Test mocking** | `Application_*.Selector.setMock(mock)` | `MySelector.replaceWith(mock)` |
| **Query methods** | *Unchanged* | *Unchanged* |

> **Warning:** Selectors and services will **no longer require Custom Metadata records** for DI binding. The binding is handled entirely in code via the static `implementationType` field and `replaceWith()` methods within each class.

---

## Selector -- Old vs New

### OLD Selector (Application Factory)

```apex
public inherited sharing class AccountSegmentTransitionSelector_AS
    extends fflib_SObjectSelector
    implements IAccountSegmentTransitionSelector_AS
{
    public static IAccountSegmentTransitionSelector_AS newInstance()
    {
        return (IAccountSegmentTransitionSelector_AS)
            ((fflib_SObjectSelector) Application_Segment.Selector.newInstance(
                Schema.AccountSegmentTransition__c.SObjectType))
                .setDataAccess(fflib_SObjectSelector.DataAccess.USER_MODE);
    }

    public static IAccountSegmentTransitionSelector_AS newElevatedInstance()
    {
        return (IAccountSegmentTransitionSelector_AS)
            ((fflib_SObjectSelector) Application_Segment.Selector.newInstance(
                Schema.AccountSegmentTransition__c.SObjectType))
                .setDataAccess(fflib_SObjectSelector.DataAccess.SYSTEM_MODE);
    }

    // Constructor, getSObjectFieldList(), getSObjectType(), query methods...
}
```

Requires registration in `Application_Segment.cls`:

```apex
public static final fflib_Application.SelectorFactory Selector =
    new fflib_Application.SelectorFactory(new Map<SObjectType, Type>{
        AccountSegmentTransition__c.SObjectType => AccountSegmentTransitionSelector_AS.class,
        // ... other selectors registered here
    });
```

Test mocking:

```apex
Application_Segment.Selector.setMock(mockSelector);
```

### NEW Selector (replaceWith, Self-Contained)

```apex
public virtual without sharing class AccountSegmentTransitionSelector_AS
    extends fflib_SObjectSelector2
    implements IAccountSegmentTransitionSelector_AS
{
    // DI managed inside the class itself
    private static System.Type implementationType = AccountSegmentTransitionSelector_AS.class;
    private static IAccountSegmentTransitionSelector_AS implementationReplacement;

    // Shared helper — avoids duplicating replacement logic
    private static IAccountSegmentTransitionSelector_AS createInstance(
            fflib_SObjectSelector2.DataAccess access) {
        IAccountSegmentTransitionSelector_AS selector = (implementationReplacement == null)
            ? (IAccountSegmentTransitionSelector_AS) implementationType.newInstance()
            : implementationReplacement;
        selector.setDataAccess(access);
        return selector;
    }

    public static IAccountSegmentTransitionSelector_AS newInstance() {
        return createInstance(fflib_SObjectSelector2.DataAccess.USER_MODE);
    }

    public static IAccountSegmentTransitionSelector_AS newElevatedInstance() {
        return createInstance(fflib_SObjectSelector2.DataAccess.SYSTEM_MODE);
    }

    // Replace with a different implementation class
    public static void replaceWith(System.Type systemType) {
        AccountSegmentTransitionSelector_AS.implementationType = systemType;
    }

    // Replace with a mock instance (for tests)
    public static void replaceWith(IAccountSegmentTransitionSelector_AS replacement) {
        AccountSegmentTransitionSelector_AS.implementationReplacement = replacement;
    }

    public AccountSegmentTransitionSelector_AS() {
        super();
    }

    public List<Schema.SObjectField> getSObjectFieldList() {
        return new List<Schema.SObjectField>{
            AccountSegmentTransition__c.Id,
            AccountSegmentTransition__c.Account__c,
            AccountSegmentTransition__c.Opportunity__c,
            AccountSegmentTransition__c.IsProcessed__c,
            AccountSegmentTransition__c.CalculatedFutureSegment__c,
            AccountSegmentTransition__c.CurrentSegment__c,
            AccountSegmentTransition__c.Status__c,
            AccountSegmentTransition__c.Reason__c,
            AccountSegmentTransition__c.CalculatedFutureOwner__c
        };
    }

    public Schema.SObjectType getSObjectType() {
        return Schema.AccountSegmentTransition__c.SObjectType;
    }

    // Query methods — UNCHANGED from old pattern
    public List<AccountSegmentTransition__c> selectById(Set<Id> idSet) {
        return (List<AccountSegmentTransition__c>) selectSObjectsById(idSet);
    }

    public List<AccountSegmentTransition__c> selectUnprocessedByOpportunityIds(Set<Id> opportunityIds) {
        return (List<AccountSegmentTransition__c>) Database.query(
            newQueryFactory()
                .setCondition('Opportunity__c IN :opportunityIds AND IsProcessed__c = FALSE')
                .toSOQL()
        );
    }
}
```

Test mocking:

```apex
AccountSegmentTransitionSelector_AS.replaceWith(mockSelector);
```

> **Info:** No Application class registration needed. No Custom Metadata record needed. The selector is fully self-contained.

---

## Service -- Old vs New

### OLD Service (Application Factory)

```apex
public with sharing class ContractService_CN {

    // Callers use:
    ((IContractService_CN) Application_Connect.Service.newInstance(
        IContractService_CN.class))
        .processActiveContracts(accountIds);
}
```

Requires registration in `Application_Connect.cls`:

```apex
public static final fflib_Application.ServiceFactory Service =
    new fflib_Application.ServiceFactory(new Map<Type, Type>{
        IContractService_CN.class => ContractService_CN.class,
        // ... other services registered here
    });
```

Test mocking:

```apex
Application_Connect.Service.setMock(IContractService_CN.class, mockService);
```

### NEW Service (replaceWith, Self-Contained)

```apex
public with sharing class ContractService_CN {

    private static System.Type implementationType = ContractServiceImpl_CN.class;
    private static IContractService_CN implementationReplacement;

    // Static facade methods delegate to the instance
    public static void processActiveContracts(Set<Id> accountIds) {
        newInstance().processActiveContracts(accountIds);
    }

    @TestVisible
    private static IContractService_CN newInstance() {
        return (implementationReplacement == null)
            ? (IContractService_CN) implementationType.newInstance()
            : implementationReplacement;
    }

    public static void replaceWith(System.Type systemType) {
        implementationType = systemType;
    }

    @TestVisible
    private static void replaceWith(IContractService_CN replacement) {
        implementationReplacement = replacement;
    }
}
```

Test mocking:

```apex
ContractService_CN.replaceWith(mockService);
```

---

## What Gets Removed

| Artifact | Action |
|---|---|
| Application_*.cls factory maps | Remove selector/service entries as classes are migrated. Delete Application class when empty. |
| Custom Metadata records for DI binding | **Delete** -- no longer needed. Bindings are now in code. |
| `Application_*.Selector.setMock()` calls in tests | Replace with `MySelector.replaceWith(mock)` |
| `Application_*.Service.setMock()` calls in tests | Replace with `MyService.replaceWith(mock)` |

---

## Migration Checklist (Per Class)

| # | Step |
|---|---|
| 1 | Change parent class from `fflib_SObjectSelector` to `fflib_SObjectSelector2` (selectors only) |
| 2 | Add static fields: `implementationType` and `implementationReplacement` |
| 3 | Update `newInstance()` / `newElevatedInstance()` to use local static fields instead of Application factory |
| 4 | Add two `replaceWith()` overloads (one for `System.Type`, one for interface instance) |
| 5 | Remove entry from `Application_*.cls` factory map |
| 6 | Delete associated Custom Metadata record |
| 7 | Update all test classes: replace `Application_*.Selector.setMock()` with `MySelector.replaceWith()` |
| 8 | Query methods -- no changes needed |

---

*Generated April 2026*
