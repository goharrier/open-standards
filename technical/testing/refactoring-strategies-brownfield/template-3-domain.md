# Domain Template

Domains wrap a **list of SObject records already in memory** and provide methods to filter, extract, and mutate them -- without touching the database.

> **Info:** Domain vs Selector:
> - **Selector** = queries records *from the database* (SOQL)
> - **Domain** = operates on records *already in memory* (no SOQL)
>
> Typical flow: a trigger fires with a list of records -> domain filters them in memory -> passes extracted IDs to a selector or service for further work.

> **Warning:** Files to create:
>
> **1** `IExpenseReports_XX.cls` -- Interface
>
> **2** `ExpenseReports_XX.cls` -- Domain class
>
> **Naming:** Domains use the **plural** SObject name (e.g., `ExpenseReports`, `Accounts`, `Contracts`).

---

## 1. Interface

```apex
public interface IExpenseReports_XX extends fflib_ISObjects {
  // Accessor
  List<Expense_Report__c> getExpenseReports();

  // Filters (return a new domain with a subset of records)
  IExpenseReports_XX selectByStatus(String status);
  IExpenseReports_XX selectByDepartment(String department);
  IExpenseReports_XX selectByAccountIds(Set<Id> accountIds);
  IExpenseReports_XX selectOverBudget(Decimal threshold);

  // Field extractors (pull IDs or values from the wrapped records)
  Set<Id> getSubmitterIds();
  Set<String> getDepartments();

  // Mutators (modify fields on the wrapped records in-place)
  IExpenseReports_XX setStatus(String status);
}
```

---

## 2. Domain Class

```apex
public inherited sharing class ExpenseReports_XX
    extends fflib_SObjects2
    implements IExpenseReports_XX
{
  // ── Factory ──────────────────────────────────────────────
  // No Application class needed. Domains are never mocked,
  // so there's no reason for factory indirection.
  // Just instantiate directly.

  public static IExpenseReports_XX newInstance(List<Expense_Report__c> records) {
    return new ExpenseReports_XX(records);
  }

  // ── Constructor ──────────────────────────────────────────

  public ExpenseReports_XX(List<Expense_Report__c> records) {
    super(records, Schema.Expense_Report__c.SObjectType);
  }

  // ── Accessor ─────────────────────────────────────────────

  public List<Expense_Report__c> getExpenseReports() {
    return (List<Expense_Report__c>) getRecords();
  }

  // ── Filters ──────────────────────────────────────────────
  // Use the built-in getRecords(SObjectField, Set<String/Id>) from fflib_SObjects2
  // instead of manual for-loops. Returns a filtered List<SObject> — wrap it
  // in a new domain to enable chaining.

  public IExpenseReports_XX selectByStatus(String status) {
    return new ExpenseReports_XX(
      getRecords(Expense_Report__c.Status__c, new Set<String>{ status })
    );
  }

  public IExpenseReports_XX selectByDepartment(String department) {
    return new ExpenseReports_XX(
      getRecords(Expense_Report__c.Department__c, new Set<String>{ department })
    );
  }

  public IExpenseReports_XX selectByAccountIds(Set<Id> accountIds) {
    return new ExpenseReports_XX(
      getRecords(Expense_Report__c.Account__c, accountIds)
    );
  }

  // For custom logic that getRecords() can't handle (e.g. comparisons),
  // fall back to a manual loop
  public IExpenseReports_XX selectOverBudget(Decimal threshold) {
    List<Expense_Report__c> result = new List<Expense_Report__c>();
    for (Expense_Report__c record : getExpenseReports()) {
      if (record.Total_Amount__c != null && record.Total_Amount__c > threshold) {
        result.add(record);
      }
    }
    return new ExpenseReports_XX(result);
  }

  // ── Field Extractors ─────────────────────────────────────
  // Use built-in getIdFieldValues() and getStringFieldValues() from fflib_SObjects2.
  // They return fflib_Ids / fflib_Strings — call .getIds() / .getStrings() for the Set.

  public Set<Id> getSubmitterIds() {
    return getIdFieldValues(Expense_Report__c.Submitter__c).getIds();
  }

  public Set<String> getDepartments() {
    return getStringFieldValues(Expense_Report__c.Department__c).getStrings();
  }

  // ── Mutators ─────────────────────────────────────────────
  // No built-in helper for this — use a manual loop.
  // Mutators modify records in-place and return 'this' for chaining.

  public IExpenseReports_XX setStatus(String status) {
    for (Expense_Report__c record : getExpenseReports()) {
      record.Status__c = status;
    }
    return this;
  }
}
```

---

## 3. Unit Tests

Domain tests are **pure unit tests** -- no mocking needed. Just create records in memory and test the domain methods.

```apex
@IsTest(IsParallel=true)
private class ExpenseReportsTest_XX {

  @IsTest
  static void itShouldFilterByStatus() {
    // given
    List<Expense_Report__c> records = new List<Expense_Report__c>{
      new Expense_Report__c(
        Id = fflib_IDGenerator.generate(Expense_Report__c.SObjectType),
        Status__c = 'Draft'
      ),
      new Expense_Report__c(
        Id = fflib_IDGenerator.generate(Expense_Report__c.SObjectType),
        Status__c = 'Pending'
      ),
      new Expense_Report__c(
        Id = fflib_IDGenerator.generate(Expense_Report__c.SObjectType),
        Status__c = 'Pending'
      )
    };

    ExpenseReports_XX domain = new ExpenseReports_XX(records);

    // when
    IExpenseReports_XX pending = domain.selectByStatus('Pending');

    // then
    Assert.areEqual(2, pending.getExpenseReports().size(),
      'Should return 2 pending reports');
  }

  @IsTest
  static void itShouldChainFilters() {
    // given
    List<Expense_Report__c> records = new List<Expense_Report__c>{
      new Expense_Report__c(
        Id = fflib_IDGenerator.generate(Expense_Report__c.SObjectType),
        Status__c = 'Pending', Department__c = 'Engineering'
      ),
      new Expense_Report__c(
        Id = fflib_IDGenerator.generate(Expense_Report__c.SObjectType),
        Status__c = 'Pending', Department__c = 'Marketing'
      ),
      new Expense_Report__c(
        Id = fflib_IDGenerator.generate(Expense_Report__c.SObjectType),
        Status__c = 'Draft', Department__c = 'Engineering'
      )
    };

    ExpenseReports_XX domain = new ExpenseReports_XX(records);

    // when — chain two filters together
    IExpenseReports_XX result = domain
      .selectByStatus('Pending')
      .selectByDepartment('Engineering');

    // then
    Assert.areEqual(1, result.getExpenseReports().size(),
      'Should return 1 pending Engineering report');
  }

  @IsTest
  static void itShouldExtractSubmitterIds() {
    // given
    Id user1 = fflib_IDGenerator.generate(User.SObjectType);
    Id user2 = fflib_IDGenerator.generate(User.SObjectType);

    List<Expense_Report__c> records = new List<Expense_Report__c>{
      new Expense_Report__c(
        Id = fflib_IDGenerator.generate(Expense_Report__c.SObjectType),
        Submitter__c = user1
      ),
      new Expense_Report__c(
        Id = fflib_IDGenerator.generate(Expense_Report__c.SObjectType),
        Submitter__c = user2
      )
    };

    ExpenseReports_XX domain = new ExpenseReports_XX(records);

    // when
    Set<Id> submitterIds = domain.getSubmitterIds();

    // then
    Assert.isTrue(submitterIds.contains(user1), 'Should contain user1');
    Assert.isTrue(submitterIds.contains(user2), 'Should contain user2');
  }

  @IsTest
  static void itShouldMutateStatusInPlace() {
    // given
    List<Expense_Report__c> records = new List<Expense_Report__c>{
      new Expense_Report__c(
        Id = fflib_IDGenerator.generate(Expense_Report__c.SObjectType),
        Status__c = 'Draft'
      )
    };

    ExpenseReports_XX domain = new ExpenseReports_XX(records);

    // when
    domain.setStatus('Submitted');

    // then — the original record is mutated (same reference)
    Assert.areEqual('Submitted', records[0].Status__c,
      'Status should be mutated in-place');
  }
}
```

> **Tip:** Domain tests need zero mocking because domains don't query the database or perform DML. You create records with `new`, pass them to the domain constructor, and assert against the results. This makes domain tests the fastest and simplest to write.

---

## Quick Reference

| What | Pattern |
| --- | --- |
| Parent class | `fflib_SObjects2` |
| Interface extends | `fflib_ISObjects` |
| Factory | `new ExpenseReports_XX(records)` or `ExpenseReports_XX.newInstance(records)` |
| Application class | **Not needed** |
| Constructor inner class | **Not needed** |
| Naming | **Plural** SObject name: `ExpenseReports_XX`, `Accounts_XX` |
| Three method types | **Filters** (return new domain), **Extractors** (return Set/Map), **Mutators** (modify in-place) |
| SOQL | **Never** -- domains only operate on in-memory records |
| Testing | Instantiate directly: `new ExpenseReports_XX(records)` |

---

*Domain Template -- April 2026*
