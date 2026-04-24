# Full Flow: Controller -> Service -> Selector + Domain + UoW

How all the layers connect end-to-end, with a complete test class.

> **Info:** **Request flow:**
>
> ```
> LWC Component
>    | calls @AuraEnabled method
> Controller -- thin, catches exceptions, resolves user context
>    | calls static facade method
> Service Facade -- one-liner delegates, DI wiring
>    | calls implementation instance
> Service Impl -- business logic, orchestration
>    | uses three tools:
>       Selector -- reads data (SOQL)
>       Domain -- filters/transforms records in memory
>       UnitOfWork -- writes data (DML)
> ```

---

## 1 Controller

Thin layer. Resolves user context, delegates to service, wraps errors.

```apex
public with sharing class ExpenseController_XX {

  @AuraEnabled(Cacheable=true)
  public static List<Expense_Report__c> getMyReports() {
    try {
      return ExpenseService_XX.getReportsForUser(UserInfo.getUserId());
    } catch (Exception e) {
      Logger.error(e.getMessage());
      Logger.saveLog();
      AuraHandledException ex = new AuraHandledException(e.getMessage());
      ex.setMessage(e.getMessage());
      throw ex;
    }
  }

  @AuraEnabled
  public static Id createReport(String reportData, String lineItems) {
    try {
      return ExpenseService_XX.createReport(
        UserInfo.getUserId(), reportData, lineItems
      );
    } catch (Exception e) {
      Logger.error(e.getMessage());
      Logger.saveLog();
      AuraHandledException ex = new AuraHandledException(e.getMessage());
      ex.setMessage(e.getMessage());
      throw ex;
    }
  }

  @AuraEnabled
  public static void submitReport(Id reportId) {
    try {
      ExpenseService_XX.submitReport(reportId);
    } catch (Exception e) {
      Logger.error(e.getMessage());
      Logger.saveLog();
      AuraHandledException ex = new AuraHandledException(e.getMessage());
      ex.setMessage(e.getMessage());
      throw ex;
    }
  }

  @AuraEnabled
  public static void approveReport(Id reportId) {
    try {
      ExpenseService_XX.approveReport(reportId);
    } catch (Exception e) {
      Logger.error(e.getMessage());
      Logger.saveLog();
      AuraHandledException ex = new AuraHandledException(e.getMessage());
      ex.setMessage(e.getMessage());
      throw ex;
    }
  }
}
```

> **Warning:** **Controller rules:** No business logic. No SOQL. No DML. The controller is a pass-through with error handling. If you find yourself writing `if` statements in a controller, that logic belongs in the service.

---

## 2 Service Facade + Interface

```apex
// Interface
public interface IExpenseService_XX {
  List<Expense_Report__c> getReportsForUser(Id userId);
  Id createReport(Id userId, String reportData, String lineItems);
  void submitReport(Id reportId);
  void approveReport(Id reportId);
}

// Facade -- static API + DI wiring
public with sharing class ExpenseService_XX {
  private static System.Type implementationType = ExpenseServiceImpl_XX.class;
  private static IExpenseService_XX implementationReplacement;

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

## 3 Service Implementation

All business logic. Uses **Selector** for queries, **Domain** for in-memory logic, **UoW** for DML.

```apex
public without sharing class ExpenseServiceImpl_XX implements IExpenseService_XX {

  public List<Expense_Report__c> getReportsForUser(Id userId) {
    return ExpenseReportSelector_XX.newElevatedInstance()
      .selectBySubmitterId(new Set<Id>{ userId });
  }

  public Id createReport(Id userId, String reportData, String lineItems) {
    Map<String, Object> data = (Map<String, Object>) JSON.deserializeUntyped(reportData);
    List<Object> lines = (List<Object>) JSON.deserializeUntyped(lineItems);

    fflib_ISObjectUnitOfWork uow = UnitOfWork_XX.newInstance();

    Expense_Report__c report = new Expense_Report__c();
    report.Submitter__c = userId;
    report.Department__c = (String) data.get('department');
    report.Status__c = 'Draft';
    uow.registerNew(report);

    for (Object lineObj : lines) {
      Map<String, Object> lineData = (Map<String, Object>) lineObj;
      Expense_Line__c line = new Expense_Line__c();
      line.Description__c = (String) lineData.get('description');
      line.Amount__c = (Decimal) lineData.get('amount');
      uow.registerNew(line, Expense_Line__c.Expense_Report__c, report);
    }

    uow.commitWork();
    return report.Id;
  }

  public void submitReport(Id reportId) {
    List<Expense_Report__c> reports = ExpenseReportSelector_XX.newElevatedInstance()
      .selectById(new Set<Id>{ reportId });

    if (reports.isEmpty()) {
      throw new AuraHandledException('Expense report not found.');
    }

    // Use domain for validation
    IExpenseReports_XX domain = new ExpenseReports_XX(reports);
    List<Expense_Report__c> draftReports = domain
      .selectByStatus('Draft')
      .getExpenseReports();

    if (draftReports.isEmpty()) {
      AuraHandledException ex = new AuraHandledException(
        'Only Draft reports can be submitted.');
      ex.setMessage('Only Draft reports can be submitted.');
      throw ex;
    }

    fflib_ISObjectUnitOfWork uow = UnitOfWork_XX.newInstance();
    // Mutate via domain, then register the changed records
    domain.selectByStatus('Draft').setStatus('Pending');
    for (Expense_Report__c report : draftReports) {
      report.Submitted_Date__c = DateTime.now();
      uow.registerDirty(report);
    }
    uow.commitWork();
  }

  public void approveReport(Id reportId) {
    List<Expense_Report__c> reports = ExpenseReportSelector_XX.newElevatedInstance()
      .selectById(new Set<Id>{ reportId });

    if (reports.isEmpty()) {
      throw new AuraHandledException('Expense report not found.');
    }

    fflib_ISObjectUnitOfWork uow = UnitOfWork_XX.newInstance();
    reports[0].Status__c = 'Approved';
    uow.registerDirty(reports[0]);
    uow.commitWork();
  }
}
```

---

## 4 UnitOfWork

```apex
public with sharing class UnitOfWork_XX {
  public final static List<SObjectType> UNIT_OF_WORK_HIERARCHY {
    get {
      if (UNIT_OF_WORK_HIERARCHY == null) {
        UNIT_OF_WORK_HIERARCHY = new List<SObjectType>{
          Expense_Report__c.SObjectType,
          Expense_Line__c.SObjectType
        };
      }
      return UNIT_OF_WORK_HIERARCHY;
    }
    private set;
  }

  private static fflib_SObjectUnitOfWork.IDML dmlReplacement;
  private static fflib_ISObjectUnitOfWork instanceReplacement;

  public static fflib_ISObjectUnitOfWork newInstance() {
    if (instanceReplacement != null) { return instanceReplacement; }
    return (dmlReplacement == null)
      ? new fflib_SObjectUnitOfWork(UNIT_OF_WORK_HIERARCHY)
      : new fflib_SObjectUnitOfWork(UNIT_OF_WORK_HIERARCHY, dmlReplacement);
  }

  @TestVisible
  private static void replaceWith(fflib_ISObjectUnitOfWork replacement) {
    UnitOfWork_XX.instanceReplacement = replacement;
  }

  @TestVisible
  private static void replaceDmlWith(fflib_SObjectUnitOfWork.IDML replacement) {
    UnitOfWork_XX.dmlReplacement = replacement;
  }
}
```

> **Info:** **UNIT_OF_WORK_HIERARCHY:** Lists SObjectTypes in parent-to-child order. UoW processes inserts top-down and deletes bottom-up, so parent records exist before children reference them.

---

## 5 Test Class -- Full Coverage

> **Warning:** **Testing strategy:**
> - **Service tests** call the **facade** (`ExpenseService_XX.submitReport(...)`), not the impl directly. The facade routes through to the impl, so both get covered.
> - **Selectors** are **mocked** in these tests -- selector coverage comes from separate, simple integration tests (see Selector template).
> - **UnitOfWork** is **mocked** -- no real DML happens.
> - **Controller tests** would mock the **service** -- testing that the @AuraEnabled wiring and error handling work.

```apex
@IsTest(IsParallel=true)
private class ExpenseServiceTest_XX {

  // -- GET: Verifies selector is called --

  @IsTest
  static void itShouldGetReportsForUser() {
    // given
    fflib_ApexMocks mocks = new fflib_ApexMocks();
    ExpenseReportSelector_XX mockSelector =
      (ExpenseReportSelector_XX) mocks.mock(ExpenseReportSelector_XX.class);

    Id userId = fflib_IDGenerator.generate(User.SObjectType);
    List<Expense_Report__c> mockReports = new List<Expense_Report__c>{
      new Expense_Report__c(
        Id = fflib_IDGenerator.generate(Expense_Report__c.SObjectType),
        Submitter__c = userId, Status__c = 'Draft'
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
    Assert.areEqual(1, result.size());
    Assert.areEqual(userId, result[0].Submitter__c);
  }

  // -- CREATE: Verifies parent + child registered to UoW --

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
      new Map<String, Object>{ 'description' => 'Flight', 'amount' => 450 }
    });

    // when
    Test.startTest();
    ExpenseService_XX.createReport(userId, reportData, lineItems);
    Test.stopTest();

    // then -- parent registered
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
    // then -- child registered with parent relationship
    ((fflib_ISObjectUnitOfWork) mocks.verify(mockUow, 1))
      .registerNew(
        (Expense_Line__c) fflib_Match.sObjectWith(
          new Map<Schema.SObjectField, Object>{
            Expense_Line__c.Description__c => 'Flight',
            Expense_Line__c.Amount__c => (Decimal) 450
          }
        ),
        Expense_Line__c.Expense_Report__c,
        (Expense_Report__c) fflib_Match.anyObject()
      );
    ((fflib_ISObjectUnitOfWork) mocks.verify(mockUow, 1)).commitWork();
  }

  // -- SUBMIT: Verifies status transition --

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

  // -- SUBMIT: Verifies business rule enforcement --

  @IsTest
  static void itShouldRejectSubmitWhenNotDraft() {
    // given
    fflib_ApexMocks mocks = new fflib_ApexMocks();
    ExpenseReportSelector_XX mockSelector =
      (ExpenseReportSelector_XX) mocks.mock(ExpenseReportSelector_XX.class);

    Id reportId = fflib_IDGenerator.generate(Expense_Report__c.SObjectType);
    Expense_Report__c mockReport = new Expense_Report__c(
      Id = reportId, Status__c = 'Approved'
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

  // -- APPROVE: Verifies status set to Approved --

  @IsTest
  static void itShouldApproveReport() {
    // given
    fflib_ApexMocks mocks = new fflib_ApexMocks();
    ExpenseReportSelector_XX mockSelector =
      (ExpenseReportSelector_XX) mocks.mock(ExpenseReportSelector_XX.class);
    fflib_ISObjectUnitOfWork mockUow =
      (fflib_ISObjectUnitOfWork) mocks.mock(fflib_ISObjectUnitOfWork.class);

    Id reportId = fflib_IDGenerator.generate(Expense_Report__c.SObjectType);
    Expense_Report__c mockReport = new Expense_Report__c(
      Id = reportId, Status__c = 'Pending'
    );

    mocks.startStubbing();
    mocks.when(mockSelector.selectById(new Set<Id>{ reportId }))
      .thenReturn(new List<Expense_Report__c>{ mockReport });
    mocks.stopStubbing();

    ExpenseReportSelector_XX.replaceWith(mockSelector);
    UnitOfWork_XX.replaceWith(mockUow);

    // when
    Test.startTest();
    ExpenseService_XX.approveReport(reportId);
    Test.stopTest();

    // then
    ((fflib_ISObjectUnitOfWork) mocks.verify(mockUow, 1))
      .registerDirty(
        (Expense_Report__c) fflib_Match.sObjectWith(
          new Map<Schema.SObjectField, Object>{
            Expense_Report__c.Status__c => 'Approved'
          }
        )
      );
    ((fflib_ISObjectUnitOfWork) mocks.verify(mockUow, 1)).commitWork();
  }

  // -- NOT FOUND: Verifies error handling --

  @IsTest
  static void itShouldThrowWhenReportNotFound() {
    // given
    fflib_ApexMocks mocks = new fflib_ApexMocks();
    ExpenseReportSelector_XX mockSelector =
      (ExpenseReportSelector_XX) mocks.mock(ExpenseReportSelector_XX.class);

    Id reportId = fflib_IDGenerator.generate(Expense_Report__c.SObjectType);

    mocks.startStubbing();
    mocks.when(mockSelector.selectById(new Set<Id>{ reportId }))
      .thenReturn(new List<Expense_Report__c>());
    mocks.stopStubbing();

    ExpenseReportSelector_XX.replaceWith(mockSelector);

    // when / then
    Test.startTest();
    try {
      ExpenseService_XX.submitReport(reportId);
      Assert.fail('Expected AuraHandledException');
    } catch (AuraHandledException ex) {
      Assert.areEqual('Expense report not found.', ex.getMessage());
    }
    Test.stopTest();
  }
}
```

---

## Architecture Summary

| Layer | Responsibility | What It Uses | How to Test |
|-------|---------------|-------------|-------------|
| **Controller** | @AuraEnabled entry point, error wrapping | Service (static calls) | Mock the service |
| **Service Facade** | Static API, DI wiring | Service Impl (via newInstance) | Not tested directly |
| **Service Impl** | Business logic, orchestration | Selector + Domain + UoW | Mock selector + UoW |
| **Selector** | SOQL queries | Database | Simple coverage test (real data); mocked in service tests |
| **Domain** | In-memory filtering & mutation | Records in memory | Direct instantiation (no mocks) |
| **UnitOfWork** | Transactional DML | Database | Mocked by service tests |
| **Trigger Action** | Detect changes, delegate | Domain + Service | Build fflib_TriggerContext manually |

## Mocking Cheat Sheet

| Mock Target | Code |
|-------------|------|
| Selector | `ExpenseReportSelector_XX.replaceWith(mockSelector)` |
| Service | `ExpenseService_XX.replaceWith(mockService)` |
| UoW (full instance) | `UnitOfWork_XX.replaceWith(mockUow)` |
| UoW (DML only) | `UnitOfWork_XX.replaceDmlWith(mockDml)` |
| Domain | No mock -- `new ExpenseReports_XX(records)` |

## Test Strategy

| Test Level | What You Call | What You Mock | Purpose |
|-----------|--------------|--------------|---------|
| Controller test | Controller (or Service facade) | Service | Verify @AuraEnabled wiring, error handling |
| Service test | **Service facade** (not impl directly) | Selector + UoW | Verify business logic, correct UoW calls |
| Selector test | Selector (with real data) | **Nothing** | Code coverage only -- logic tested via service tests |
| Domain test | Domain (with in-memory records) | **Nothing** | Pure unit test -- verify filtering, extraction, mutation |
| Trigger action test | Trigger action via fflib_TriggerContext | Service | Verify field change detection, correct delegation |

---

*Full Flow Template -- April 2026*
