# Selector Template

Selectors are the **only layer that talks to the database**. They encapsulate all SOQL queries behind a mockable interface.

> **Info:** What a selector does:
> - Defines which fields are queried by default (`getSObjectFieldList`)
> - Provides named query methods (`selectByX`, `selectWhereY`)
> - Manages its own DI via `replaceWith` -- no Application class or Custom Metadata needed

> **Warning:** Files to create:
> 1. `IExpenseReportSelector_XX.cls` -- Interface
> 2. `ExpenseReportSelector_XX.cls` -- Implementation

---

## 1. Interface

Defines the query methods your selector exposes. Other layers depend on this, not the concrete class.

```apex
public interface IExpenseReportSelector_XX extends fflib_ISObjectSelector {
  List<Expense_Report__c> selectById(Set<Id> idSet);
  List<Expense_Report__c> selectBySubmitterId(Set<Id> submitterIds);
  List<Expense_Report__c> selectPendingByDepartment(String department);
}
```

---

## 2. Selector Class

```apex
public virtual without sharing class ExpenseReportSelector_XX
    extends fflib_SObjectSelector2
    implements IExpenseReportSelector_XX
{
  // ── DI Wiring ────────────────────────────────────────────
  // These two static fields are how replaceWith works.
  // In production: implementationType creates a real instance.
  // In tests: implementationReplacement injects a mock.

  private static System.Type implementationType = ExpenseReportSelector_XX.class;
  private static IExpenseReportSelector_XX implementationReplacement;

  // ── Factory ──────────────────────────────────────────────
  // createInstance() is a private helper so that newInstance()
  // and newElevatedInstance() don't duplicate the replacement logic.

  private static IExpenseReportSelector_XX createInstance(
      fflib_SObjectSelector2.DataAccess access) {
    IExpenseReportSelector_XX selector = (implementationReplacement == null)
      ? (IExpenseReportSelector_XX) implementationType.newInstance()
      : implementationReplacement;
    selector.setDataAccess(access);
    return selector;
  }

  // USER_MODE = respects sharing rules and field-level security
  public static IExpenseReportSelector_XX newInstance() {
    return createInstance(fflib_SObjectSelector2.DataAccess.USER_MODE);
  }

  // SYSTEM_MODE = bypasses sharing (use in service impl layer)
  public static IExpenseReportSelector_XX newElevatedInstance() {
    return createInstance(fflib_SObjectSelector2.DataAccess.SYSTEM_MODE);
  }

  // Called in production to swap the implementation class
  public static void replaceWith(System.Type systemType) {
    ExpenseReportSelector_XX.implementationType = systemType;
  }

  // Called in tests to inject a mock
  public static void replaceWith(IExpenseReportSelector_XX replacement) {
    ExpenseReportSelector_XX.implementationReplacement = replacement;
  }

  // ── Selector Configuration ───────────────────────────────

  public ExpenseReportSelector_XX() {
    super();
  }

  // Default fields included in every query from this selector
  public List<Schema.SObjectField> getSObjectFieldList() {
    return new List<Schema.SObjectField>{
      Expense_Report__c.Id,
      Expense_Report__c.Name,
      Expense_Report__c.Submitter__c,
      Expense_Report__c.Department__c,
      Expense_Report__c.Status__c,
      Expense_Report__c.Total_Amount__c,
      Expense_Report__c.Submitted_Date__c
    };
  }

  public Schema.SObjectType getSObjectType() {
    return Expense_Report__c.SObjectType;
  }

  // ── Query Methods ────────────────────────────────────────

  public virtual List<Expense_Report__c> selectById(Set<Id> idSet) {
    return (List<Expense_Report__c>) selectSObjectsById(idSet);
  }

  public virtual List<Expense_Report__c> selectBySubmitterId(Set<Id> submitterIds) {
    return (List<Expense_Report__c>) Database.query(
      newQueryFactory()
        .setCondition('Submitter__c IN :submitterIds')
        .setOrdering(Expense_Report__c.Submitted_Date__c,
                     fflib_QueryFactory.SortOrder.DESCENDING)
        .toSOQL()
    );
  }

  public virtual List<Expense_Report__c> selectPendingByDepartment(String department) {
    return (List<Expense_Report__c>) Database.query(
      newQueryFactory()
        .setCondition('Department__c = :department AND Status__c = \'Pending\'')
        .toSOQL()
    );
  }
}
```

---

## 3. Selector Test (Coverage Only)

> **Warning:** You do **not** mock a selector to test itself -- that would be circular and test nothing. Selector tests are simple **integration tests** that insert real data and call the query method. Their purpose is **code coverage**, not logic verification. The real testing of selector behaviour happens indirectly through **service tests** that mock the selector.

```apex
@IsTest
private class ExpenseReportSelectorTest_XX {

  @TestSetup
  static void setup() {
    // Insert real test data — selector tests hit the database
    insert new Expense_Report__c(
      Name = 'Test Report',
      Department__c = 'Engineering',
      Status__c = 'Pending'
    );
  }

  @IsTest
  static void itShouldSelectPendingByDepartment() {
    // This test exists for code coverage.
    // The actual selector logic is tested indirectly via service tests.
    Test.startTest();
    List<Expense_Report__c> result =
      ExpenseReportSelector_XX.newInstance()
        .selectPendingByDepartment('Engineering');
    Test.stopTest();

    Assert.isFalse(result.isEmpty(), 'Should return at least one record');
  }

  @IsTest
  static void itShouldSelectBySubmitterId() {
    Test.startTest();
    List<Expense_Report__c> result =
      ExpenseReportSelector_XX.newInstance()
        .selectBySubmitterId(new Set<Id>{ UserInfo.getUserId() });
    Test.stopTest();

    // Just verify it runs without error — coverage is the goal
    Assert.isNotNull(result);
  }
}
```

---

## 4. How Selectors Are Mocked (in Service Tests)

You **never** mock a selector to test the selector. You mock it when testing the **service** that uses it. This is shown in the Service template, but here's the pattern:

```apex
// Inside a SERVICE test — mock the selector so the service test has no SOQL
fflib_ApexMocks mocks = new fflib_ApexMocks();
ExpenseReportSelector_XX mockSelector =
  (ExpenseReportSelector_XX) mocks.mock(ExpenseReportSelector_XX.class);

mocks.startStubbing();
mocks.when(mockSelector.selectBySubmitterId(new Set<Id>{ userId }))
  .thenReturn(mockReports);
mocks.stopStubbing();

// Inject the mock — service will get this instead of the real selector
ExpenseReportSelector_XX.replaceWith(mockSelector);

// Now call the SERVICE (not the selector) — it uses the mock internally
List<Expense_Report__c> result = ExpenseService_XX.getReportsForUser(userId);
```

> **Tip:** Selector mocking happens in **service tests**, not selector tests. The service test calls the service facade, the facade calls the impl, and the impl calls the selector -- which returns mock data instead of hitting the database.

---

## Quick Reference

| What | Pattern |
| --- | --- |
| Parent class | `fflib_SObjectSelector2` |
| DI fields | `implementationType` + `implementationReplacement` |
| Shared helper | `createInstance(DataAccess)` avoids duplicating replacement logic |
| Public factory | `newInstance()` (user mode) + `newElevatedInstance()` (system mode) |
| Mock injection | `ExpenseReportSelector_XX.replaceWith(mockSelector)` |
| Custom Metadata | **Not needed** |
| Application class | **Not needed** |

*Selector Template -- April 2026*
