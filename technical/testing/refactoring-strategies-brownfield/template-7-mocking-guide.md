# Mocking Guide

How to mock selectors, services, and UnitOfWork in different scenarios. Includes good/bad examples and common mistakes.

---

## 1. What Gets Mocked and What Doesn't

| Component | Mock? | Why |
|-----------|-------|-----|
| **Selector** | Yes (in service tests) | Avoids real SOQL. You control exactly what data the service sees. |
| **Service** | Yes (in controller/trigger action tests) | Isolates the caller from business logic. |
| **UnitOfWork** | Yes (in service tests) | Avoids real DML. You verify the right records were registered. |
| **Domain** | **Never** | Domains are plain objects. They receive data from mocked selectors and run real logic. No DI to replace. |

---

## 2. Basic Pattern — Mock One Selector + UoW

The most common scenario. Service queries one selector, does logic, writes via UoW.

```apex
@IsTest
static void itShouldApproveReport() {
  // ── GIVEN ────────────────────────────────────────────────
  fflib_ApexMocks mocks = new fflib_ApexMocks();

  // Mock the selector
  ExpenseReportSelector_XX mockSelector =
    (ExpenseReportSelector_XX) mocks.mock(ExpenseReportSelector_XX.class);

  // Mock the UoW
  fflib_ISObjectUnitOfWork mockUow =
    (fflib_ISObjectUnitOfWork) mocks.mock(fflib_ISObjectUnitOfWork.class);

  // Set up test data (no DML — just in-memory records)
  Id reportId = fflib_IDGenerator.generate(Expense_Report__c.SObjectType);
  Expense_Report__c mockReport = new Expense_Report__c(
    Id = reportId, Status__c = 'Pending'
  );

  // Stub the selector to return our mock data
  mocks.startStubbing();
  mocks.when(mockSelector.selectById(new Set<Id>{ reportId }))
    .thenReturn(new List<Expense_Report__c>{ mockReport });
  mocks.stopStubbing();

  // Inject mocks
  ExpenseReportSelector_XX.replaceWith(mockSelector);
  UnitOfWork_XX.replaceWith(mockUow);

  // ── WHEN ─────────────────────────────────────────────────
  Test.startTest();
  ExpenseService_XX.approveReport(reportId);
  Test.stopTest();

  // ── THEN ─────────────────────────────────────────────────
  // Verify the UoW received the right record with the right field values
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
```

---

## 3. Multiple Selectors

When a service method queries multiple selectors, mock them all in the same test.

```apex
@IsTest
static void itShouldSubmitWithApproverLookup() {
  fflib_ApexMocks mocks = new fflib_ApexMocks();

  // Mock TWO selectors
  ExpenseReportSelector_XX mockReportSelector =
    (ExpenseReportSelector_XX) mocks.mock(ExpenseReportSelector_XX.class);
  UserSelector_XX mockUserSelector =
    (UserSelector_XX) mocks.mock(UserSelector_XX.class);
  fflib_ISObjectUnitOfWork mockUow =
    (fflib_ISObjectUnitOfWork) mocks.mock(fflib_ISObjectUnitOfWork.class);

  Id reportId = fflib_IDGenerator.generate(Expense_Report__c.SObjectType);
  Id managerId = fflib_IDGenerator.generate(User.SObjectType);

  Expense_Report__c mockReport = new Expense_Report__c(
    Id = reportId, Status__c = 'Draft', Submitter__c = managerId
  );
  User mockManager = new User(Id = managerId, ManagerId = managerId);

  // Stub BOTH selectors
  mocks.startStubbing();
  mocks.when(mockReportSelector.selectById(new Set<Id>{ reportId }))
    .thenReturn(new List<Expense_Report__c>{ mockReport });
  mocks.when(mockUserSelector.selectByIds(new Set<Id>{ managerId }))
    .thenReturn(new List<User>{ mockManager });
  mocks.stopStubbing();

  // Inject BOTH mocks
  ExpenseReportSelector_XX.replaceWith(mockReportSelector);
  UserSelector_XX.replaceWith(mockUserSelector);
  UnitOfWork_XX.replaceWith(mockUow);

  // when
  Test.startTest();
  ExpenseService_XX.submitReport(reportId);
  Test.stopTest();

  // then — verify both selectors were used and UoW got the right data
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
```

> **Pattern:** One `fflib_ApexMocks` instance, one `startStubbing/stopStubbing` block, all stubs inside. Each selector gets its own `replaceWith` call.

---

## 4. Mocking a Service (from Controller or Trigger Action)

When testing a controller, mock the service so you never touch the impl/selector/UoW.

```apex
@IsTest
static void itShouldReturnBudgetsFromController() {
  fflib_ApexMocks mocks = new fflib_ApexMocks();

  // Mock the SERVICE (not the selector)
  IExpenseService_XX mockService =
    (IExpenseService_XX) mocks.mock(IExpenseService_XX.class);

  Id userId = fflib_IDGenerator.generate(User.SObjectType);
  List<Expense_Report__c> mockReports = new List<Expense_Report__c>{
    new Expense_Report__c(
      Id = fflib_IDGenerator.generate(Expense_Report__c.SObjectType),
      Status__c = 'Draft'
    )
  };

  mocks.startStubbing();
  mocks.when(mockService.getReportsForUser(userId))
    .thenReturn(mockReports);
  mocks.stopStubbing();

  // Inject at the service level — everything below is bypassed
  ExpenseService_XX.replaceWith(mockService);

  // when — call through the service facade
  Test.startTest();
  List<Expense_Report__c> result = ExpenseService_XX.getReportsForUser(userId);
  Test.stopTest();

  // then
  Assert.areEqual(1, result.size());
}
```

> **When to mock the service:** Controller tests and trigger action tests. You're testing the *caller*, not the business logic.

---

## 5. Verifying UoW — Parent + Child Records

When a service creates a parent record with child records linked via a relationship field.

```apex
// Verify parent was registered
((fflib_ISObjectUnitOfWork) mocks.verify(mockUow, 1))
  .registerNew(
    (Expense_Report__c) fflib_Match.sObjectWith(
      new Map<Schema.SObjectField, Object>{
        Expense_Report__c.Department__c => 'Engineering',
        Expense_Report__c.Status__c => 'Draft'
      }
    )
  );

// Verify child was registered WITH parent relationship
// The 3-argument registerNew links child to parent before commit
((fflib_ISObjectUnitOfWork) mocks.verify(mockUow, 1))
  .registerNew(
    (Expense_Line__c) fflib_Match.sObjectWith(
      new Map<Schema.SObjectField, Object>{
        Expense_Line__c.Description__c => 'Flight',
        Expense_Line__c.Amount__c => (Decimal) 450
      }
    ),
    Expense_Line__c.Expense_Report__c,      // relationship field
    (Expense_Report__c) fflib_Match.anyObject()  // parent record
  );

// Verify commit happened exactly once
((fflib_ISObjectUnitOfWork) mocks.verify(mockUow, 1)).commitWork();
```

---

## 6. Verifying UoW — Delete + Create (Replace Pattern)

When a service deletes existing children and creates new ones (e.g., replacing line items on an update).

```apex
// Verify old children were deleted
((fflib_ISObjectUnitOfWork) mocks.verify(mockUow, 1))
  .registerDeleted(
    (Expense_Line__c) fflib_Match.sObjectWith(
      new Map<Schema.SObjectField, Object>{
        Expense_Line__c.Id => existingLineId
      }
    )
  );

// Verify new children were created
((fflib_ISObjectUnitOfWork) mocks.verify(mockUow, 1))
  .registerNew(
    (Expense_Line__c) fflib_Match.sObjectWith(
      new Map<Schema.SObjectField, Object>{
        Expense_Line__c.Description__c => 'New Flight'
      }
    )
  );

// Verify parent was updated
((fflib_ISObjectUnitOfWork) mocks.verify(mockUow, 1))
  .registerDirty(
    (Expense_Report__c) fflib_Match.sObjectWith(
      new Map<Schema.SObjectField, Object>{
        Expense_Report__c.Id => reportId
      }
    )
  );

((fflib_ISObjectUnitOfWork) mocks.verify(mockUow, 1)).commitWork();
```

---

## 7. Mocking Multiple Services

When a service impl calls another service (e.g., notification after approval).

```apex
@IsTest
static void itShouldApproveAndNotify() {
  fflib_ApexMocks mocks = new fflib_ApexMocks();

  ExpenseReportSelector_XX mockSelector =
    (ExpenseReportSelector_XX) mocks.mock(ExpenseReportSelector_XX.class);
  fflib_ISObjectUnitOfWork mockUow =
    (fflib_ISObjectUnitOfWork) mocks.mock(fflib_ISObjectUnitOfWork.class);

  // Mock ANOTHER service that gets called internally
  INotificationService_XX mockNotifService =
    (INotificationService_XX) mocks.mock(INotificationService_XX.class);

  Id reportId = fflib_IDGenerator.generate(Expense_Report__c.SObjectType);

  mocks.startStubbing();
  mocks.when(mockSelector.selectById(new Set<Id>{ reportId }))
    .thenReturn(new List<Expense_Report__c>{
      new Expense_Report__c(Id = reportId, Status__c = 'Pending')
    });
  mocks.stopStubbing();

  ExpenseReportSelector_XX.replaceWith(mockSelector);
  UnitOfWork_XX.replaceWith(mockUow);
  NotificationService_XX.replaceWith(mockNotifService);

  // when
  Test.startTest();
  ExpenseService_XX.approveReport(reportId);
  Test.stopTest();

  // then — verify the notification service was called
  ((INotificationService_XX) mocks.verify(mockNotifService, 1))
    .sendApprovalNotification(reportId);
  ((fflib_ISObjectUnitOfWork) mocks.verify(mockUow, 1)).commitWork();
}
```

---

## 8. Testing Error / Exception Paths

Mock the selector to return data that triggers the error path.

```apex
// ── Not Found ──────────────────────────────────────────────
@IsTest
static void itShouldThrowWhenNotFound() {
  fflib_ApexMocks mocks = new fflib_ApexMocks();
  ExpenseReportSelector_XX mockSelector =
    (ExpenseReportSelector_XX) mocks.mock(ExpenseReportSelector_XX.class);

  Id reportId = fflib_IDGenerator.generate(Expense_Report__c.SObjectType);

  mocks.startStubbing();
  // Return EMPTY list — triggers "not found" error
  mocks.when(mockSelector.selectById(new Set<Id>{ reportId }))
    .thenReturn(new List<Expense_Report__c>());
  mocks.stopStubbing();

  ExpenseReportSelector_XX.replaceWith(mockSelector);

  Test.startTest();
  try {
    ExpenseService_XX.submitReport(reportId);
    Assert.fail('Expected AuraHandledException');
  } catch (AuraHandledException ex) {
    Assert.areEqual('Expense report not found.', ex.getMessage());
  }
  Test.stopTest();
}

// ── Invalid State ──────────────────────────────────────────
@IsTest
static void itShouldThrowWhenNotDraft() {
  fflib_ApexMocks mocks = new fflib_ApexMocks();
  ExpenseReportSelector_XX mockSelector =
    (ExpenseReportSelector_XX) mocks.mock(ExpenseReportSelector_XX.class);

  Id reportId = fflib_IDGenerator.generate(Expense_Report__c.SObjectType);

  mocks.startStubbing();
  // Return record with WRONG status — triggers validation error
  mocks.when(mockSelector.selectById(new Set<Id>{ reportId }))
    .thenReturn(new List<Expense_Report__c>{
      new Expense_Report__c(Id = reportId, Status__c = 'Approved')
    });
  mocks.stopStubbing();

  ExpenseReportSelector_XX.replaceWith(mockSelector);

  Test.startTest();
  try {
    ExpenseService_XX.submitReport(reportId);
    Assert.fail('Expected AuraHandledException');
  } catch (AuraHandledException ex) {
    Assert.areEqual('Only Draft reports can be submitted.', ex.getMessage());
  }
  Test.stopTest();
}
```

---

## 9. Common Mistakes

> **BAD** Mocking the selector to test the selector

```apex
// This tests NOTHING — you're verifying that your mock returns what you told it to
ExpenseReportSelector_XX mockSelector = (ExpenseReportSelector_XX) mocks.mock(...);
mocks.when(mockSelector.selectById(ids)).thenReturn(fakeData);
ExpenseReportSelector_XX.replaceWith(mockSelector);
List<Expense_Report__c> result = ExpenseReportSelector_XX.newInstance().selectById(ids);
Assert.areEqual(fakeData, result);  // Of course it equals — you set it up that way
```

> **GOOD** Selector tests use real data for coverage

```apex
// Selector test — real data, no mocking
@TestSetup
static void setup() {
  insert new Expense_Report__c(Name = 'Test', Status__c = 'Pending', Department__c = 'Eng');
}

@IsTest
static void itShouldSelectPendingByDepartment() {
  List<Expense_Report__c> result =
    ExpenseReportSelector_XX.newInstance().selectPendingByDepartment('Eng');
  Assert.isFalse(result.isEmpty());
}
```

---

> **BAD** Testing the impl class directly

```apex
// Bypasses the facade — doesn't test the DI wiring or cover the facade code
ExpenseServiceImpl_XX impl = new ExpenseServiceImpl_XX();
impl.submitReport(reportId);
```

> **GOOD** Test through the facade

```apex
// Goes through the facade — covers both facade and impl
ExpenseService_XX.submitReport(reportId);
```

---

> **BAD** Forgetting to stub the selector (returns null)

```apex
// Mock created but never stubbed — selectById returns null, causing NPE
ExpenseReportSelector_XX mockSelector = (ExpenseReportSelector_XX) mocks.mock(...);
ExpenseReportSelector_XX.replaceWith(mockSelector);
// MISSING: mocks.startStubbing() / mocks.when(...) / mocks.stopStubbing()
```

> **GOOD** Always stub the methods your service will call

```apex
mocks.startStubbing();
mocks.when(mockSelector.selectById(new Set<Id>{ reportId }))
  .thenReturn(new List<Expense_Report__c>{ mockReport });
mocks.stopStubbing();
```

---

> **BAD** Verifying with wrong count

```apex
// Verifying commitWork was called 2 times when the service only commits once
((fflib_ISObjectUnitOfWork) mocks.verify(mockUow, 2)).commitWork();
// This PASSES silently if commitWork was never called (ApexMocks quirk with 0 calls)
```

> **GOOD** Verify with the exact expected count

```apex
// One commit per service method call
((fflib_ISObjectUnitOfWork) mocks.verify(mockUow, 1)).commitWork();
```

---

## 10. Cheat Sheet

| Scenario | What to Mock | How to Inject |
|----------|-------------|---------------|
| Service test (basic) | 1 selector + UoW | `Selector.replaceWith(mock)` / `UnitOfWork_XX.replaceWith(mockUow)` |
| Service test (multiple queries) | Multiple selectors + UoW | Each selector gets its own `replaceWith` |
| Service test (calls another service) | Selectors + UoW + other service | `OtherService_XX.replaceWith(mockOther)` |
| Controller test | Service only | `MyService_XX.replaceWith(mockService)` |
| Trigger action test | Service (+ build TriggerContext) | `MyService_XX.replaceWith(mockService)` |
| Selector test | **Nothing** | Insert real data, call selector, assert not empty |
| Domain test | **Nothing** | `new MyDomain_XX(records)` with in-memory data |
| Error path test | Selector (return empty or wrong-state data) | Same as basic, but stub returns trigger the error |

## Verify Patterns Quick Reference

| UoW Action | Verify Pattern |
|------------|---------------|
| Insert (no parent) | `mocks.verify(mockUow, 1)).registerNew(sObjectMatch)` |
| Insert (with parent) | `mocks.verify(mockUow, 1)).registerNew(sObjectMatch, relationshipField, parentMatch)` |
| Update | `mocks.verify(mockUow, 1)).registerDirty(sObjectMatch)` |
| Delete | `mocks.verify(mockUow, 1)).registerDeleted(sObjectMatch)` |
| Commit | `mocks.verify(mockUow, 1)).commitWork()` |
| Verify NOT called | `mocks.verify(mockUow, **0**)).commitWork()` |

---

*Mocking Guide -- April 2026*
