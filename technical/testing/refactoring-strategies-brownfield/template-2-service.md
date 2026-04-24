# Service Template

Services contain **all business logic**. They follow a **3-file pattern**: an interface defines the contract, a facade provides the static API + DI wiring, and an implementation holds the actual logic.

> **Info:** Why 3 files?
> - **Interface** -- defines the contract. Other layers depend on this.
> - **Facade** -- static entry point (`ExpenseService_XX.submit(...)`). Contains zero business logic. Handles DI wiring.
> - **Implementation** -- all business logic lives here. Uses selectors for queries, UoW for DML.

> **Warning:** Files to create:
> 1. `IExpenseService_XX.cls` -- Interface
> 2. `ExpenseService_XX.cls` -- Facade (static API + DI)
> 3. `ExpenseServiceImpl_XX.cls` -- Implementation (business logic)

---

## 1. Interface

```apex
public interface IExpenseService_XX {
  List<Expense_Report__c> getReportsForUser(Id userId);
  Id createReport(Id userId, String reportData, String lineItems);
  void submitReport(Id reportId);
  void approveReport(Id reportId);
}
```

---

## 2. Service Facade

Every public method is a **one-liner** that delegates to `newInstance()`. No logic here.

```apex
public with sharing class ExpenseService_XX {
  private static System.Type implementationType = ExpenseServiceImpl_XX.class;
  private static IExpenseService_XX implementationReplacement;

  // ── Static API (one-liner delegates) ─────────────────────

  public static List<Expense_Report__c> getReportsForUser(Id userId) {
    return newInstance().getReportsForUser(userId);
  }

  public static Id createReport(Id userId, String reportData, String lineItems) {
    return newInstance().createReport(userId, reportData, lineItems);
  }

  public static void submitReport(Id reportId) {
    newInstance().submitReport(reportId);
  }

  public static void approveReport(Id reportId) {
    newInstance().approveReport(reportId);
  }

  // ── DI Wiring ────────────────────────────────────────────

  public static void replaceWith(System.Type systemType) {
    ExpenseService_XX.implementationType = systemType;
  }

  @TestVisible
  private static void replaceWith(IExpenseService_XX replacement) {
    ExpenseService_XX.implementationReplacement = replacement;
  }

  @TestVisible
  private static IExpenseService_XX newInstance() {
    return (implementationReplacement == null)
      ? (IExpenseService_XX) implementationType.newInstance()
      : implementationReplacement;
  }
}
```

## 3. Service Implementation

This is where all business logic lives. It calls **selectors** for data and **UnitOfWork** for DML.

```apex
public without sharing class ExpenseServiceImpl_XX implements IExpenseService_XX {

  public List<Expense_Report__c> getReportsForUser(Id userId) {
    // Selector handles the SOQL
    return ExpenseReportSelector_XX.newElevatedInstance()
      .selectBySubmitterId(new Set<Id>{ userId });
  }

  public Id createReport(Id userId, String reportData, String lineItems) {
    Map<String, Object> data = (Map<String, Object>) JSON.deserializeUntyped(reportData);
    List<Object> lines = (List<Object>) JSON.deserializeUntyped(lineItems);

    // UnitOfWork handles all DML in one transaction
    fflib_ISObjectUnitOfWork uow = UnitOfWork_XX.newInstance();

    Expense_Report__c report = new Expense_Report__c();
    report.Submitter__c = userId;
    report.Department__c = (String) data.get('department');
    report.Status__c = 'Draft';
    uow.registerNew(report);

    // Child records linked via relationship field
    for (Object lineObj : lines) {
      Map<String, Object> lineData = (Map<String, Object>) lineObj;
      Expense_Line__c line = new Expense_Line__c();
      line.Description__c = (String) lineData.get('description');
      line.Amount__c = (Decimal) lineData.get('amount');
      // Links line to report (parent not yet committed, UoW handles this)
      uow.registerNew(line, Expense_Line__c.Expense_Report__c, report);
    }

    uow.commitWork();
    return report.Id;
  }

  public void submitReport(Id reportId) {
    // Query the record
    List<Expense_Report__c> reports = ExpenseReportSelector_XX.newElevatedInstance()
      .selectById(new Set<Id>{ reportId });

    if (reports.isEmpty()) {
      throw new AuraHandledException('Expense report not found.');
    }

    // Business rule: only Draft reports can be submitted
    Expense_Report__c report = reports[0];
    if (report.Status__c != 'Draft') {
      AuraHandledException ex = new AuraHandledException('Only Draft reports can be submitted.');
      ex.setMessage('Only Draft reports can be submitted.');
      throw ex;
    }

    // Update and commit
    fflib_ISObjectUnitOfWork uow = UnitOfWork_XX.newInstance();
    report.Status__c = 'Pending';
    report.Submitted_Date__c = DateTime.now();
    uow.registerDirty(report);
    uow.commitWork();
  }

  public void approveReport(Id reportId) {
    List<Expense_Report__c> reports = ExpenseReportSelector_XX.newElevatedInstance()
      .selectById(new Set<Id>{ reportId });

    if (reports.isEmpty()) {
      throw new AuraHandledException('Expense report not found.');
    }

    fflib_ISObjectUnitOfWork uow = UnitOfWork_XX.newInstance();
    Expense_Report__c report = reports[0];
    report.Status__c = 'Approved';
    uow.registerDirty(report);
    uow.commitWork();
  }
}
```

## 4. Unit Tests

> **Warning:** Service tests call the **facade** (`ExpenseService_XX.submitReport(...)`), not the impl class directly. The facade routes to the impl via `newInstance()`, so you get coverage of both. You mock the **selector** (no SOQL) and **UnitOfWork** (no DML) underneath.

```apex
@IsTest(IsParallel=true)
private class ExpenseServiceTest_XX {

  // ── Test: Query delegated to selector ────────────────────

  @IsTest
  static void itShouldGetReportsViaSelector() {
    // given
    fflib_ApexMocks mocks = new fflib_ApexMocks();
    ExpenseReportSelector_XX mockSelector =
      (ExpenseReportSelector_XX) mocks.mock(ExpenseReportSelector_XX.class);

    Id userId = fflib_IDGenerator.generate(User.SObjectType);
    List<Expense_Report__c> mockReports = new List<Expense_Report__c>{
      new Expense_Report__c(
        Id = fflib_IDGenerator.generate(Expense_Report__c.SObjectType),
        Submitter__c = userId,
        Status__c = 'Draft'
      )
    };

    mocks.startStubbing();
    mocks.when(mockSelector.selectBySubmitterId(new Set<Id>{ userId }))
      .thenReturn(mockReports);
    mocks.stopStubbing();

    ExpenseReportSelector_XX.replaceWith(mockSelector);

    // when
    Test.startTest();
    List<Expense_Report__c> result = ExpenseService_XX.getReportsForUser(userId);
    Test.stopTest();

    // then
    Assert.areEqual(1, result.size(), 'Should return one report');
  }

  // ── Test: Create with child records ──────────────────────

  @IsTest
  static void itShouldCreateReportWithLineItems() {
    // given
    fflib_ApexMocks mocks = new fflib_ApexMocks();
    fflib_ISObjectUnitOfWork mockUow =
      (fflib_ISObjectUnitOfWork) mocks.mock(fflib_ISObjectUnitOfWork.class);

    UnitOfWork_XX.replaceWith(mockUow);

    Id userId = fflib_IDGenerator.generate(User.SObjectType);
    String reportData = JSON.serialize(new Map<String, Object>{
      'department' => 'Engineering'
    });
    String lineItems = JSON.serialize(new List<Object>{
      new Map<String, Object>{ 'description' => 'Flight', 'amount' => 450 },
      new Map<String, Object>{ 'description' => 'Hotel',  'amount' => 200 }
    });

    // when
    Test.startTest();
    ExpenseService_XX.createReport(userId, reportData, lineItems);
    Test.stopTest();

    // then — verify the parent record was registered
    ((fflib_ISObjectUnitOfWork) mocks.verify(mockUow, 1))
      .registerNew(
        (Expense_Report__c) fflib_Match.sObjectWith(
          new Map<Schema.SObjectField, Object>{
            Expense_Report__c.Submitter__c => userId,
            Expense_Report__c.Department__c => 'Engineering',
            Expense_Report__c.Status__c => 'Draft'
          }
        )
      );

    // then — verify child records were registered with parent relationship
    ((fflib_ISObjectUnitOfWork) mocks.verify(mockUow, 2))
      .registerNew(
        (Expense_Line__c) fflib_Match.anyObject(),
        Expense_Line__c.Expense_Report__c,
        (Expense_Report__c) fflib_Match.anyObject()
      );

    ((fflib_ISObjectUnitOfWork) mocks.verify(mockUow, 1)).commitWork();
  }

  // ── Test: Submit changes status ──────────────────────────

  @IsTest
  static void itShouldSubmitDraftReport() {
    // given
    fflib_ApexMocks mocks = new fflib_ApexMocks();
    ExpenseReportSelector_XX mockSelector =
      (ExpenseReportSelector_XX) mocks.mock(ExpenseReportSelector_XX.class);
    fflib_ISObjectUnitOfWork mockUow =
      (fflib_ISObjectUnitOfWork) mocks.mock(fflib_ISObjectUnitOfWork.class);

    Id reportId = fflib_IDGenerator.generate(Expense_Report__c.SObjectType);
    Expense_Report__c mockReport = new Expense_Report__c(
      Id = reportId, Status__c = 'Draft'
    );

    mocks.startStubbing();
    mocks.when(mockSelector.selectById(new Set<Id>{ reportId }))
      .thenReturn(new List<Expense_Report__c>{ mockReport });
    mocks.stopStubbing();

    ExpenseReportSelector_XX.replaceWith(mockSelector);
    UnitOfWork_XX.replaceWith(mockUow);

    // when
    Test.startTest();
    ExpenseService_XX.submitReport(reportId);
    Test.stopTest();

    // then
    ((fflib_ISObjectUnitOfWork) mocks.verify(mockUow, 1))
      .registerDirty(
        (Expense_Report__c) fflib_Match.sObjectWith(
          new Map<Schema.SObjectField, Object>{
            Expense_Report__c.Status__c => 'Pending'
          }
        )
      );
    ((fflib_ISObjectUnitOfWork) mocks.verify(mockUow, 1)).commitWork();
  }

  // ── Test: Business rule enforcement ──────────────────────

  @IsTest
  static void itShouldRejectSubmitWhenNotDraft() {
    // given
    fflib_ApexMocks mocks = new fflib_ApexMocks();
    ExpenseReportSelector_XX mockSelector =
      (ExpenseReportSelector_XX) mocks.mock(ExpenseReportSelector_XX.class);

    Id reportId = fflib_IDGenerator.generate(Expense_Report__c.SObjectType);
    Expense_Report__c mockReport = new Expense_Report__c(
      Id = reportId, Status__c = 'Approved'  // Not Draft
    );

    mocks.startStubbing();
    mocks.when(mockSelector.selectById(new Set<Id>{ reportId }))
      .thenReturn(new List<Expense_Report__c>{ mockReport });
    mocks.stopStubbing();

    ExpenseReportSelector_XX.replaceWith(mockSelector);

    // when / then
    Test.startTest();
    try {
      ExpenseService_XX.submitReport(reportId);
      Assert.fail('Expected AuraHandledException');
    } catch (AuraHandledException ex) {
      Assert.areEqual('Only Draft reports can be submitted.', ex.getMessage());
    }
    Test.stopTest();
  }
}
```

## Quick Reference

| What | Pattern |
| --- | --- |
| Facade sharing | `with sharing` (enforces record access) |
| Impl sharing | `without sharing` (service needs full access internally) |
| Facade methods | One-liner static delegates -- zero business logic |
| Impl methods | All business logic: validation, orchestration, data transformation |
| Data access | Via selectors: `MySelector_XX.newElevatedInstance()` |
| DML | Via UnitOfWork: `UnitOfWork_XX.newInstance()` |
| Mock service | `ExpenseService_XX.replaceWith(mockService)` |
| Custom Metadata | **Not needed** |

---

*Service Template -- April 2026*
