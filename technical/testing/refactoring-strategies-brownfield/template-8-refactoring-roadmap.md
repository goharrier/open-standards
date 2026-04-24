# Refactoring Roadmap

A phased approach to migrating legacy Apex code to the fflib separation-of-concerns pattern. Each phase is independently deployable and adds value without requiring the next phase to be complete.

> **Tip:** Core principle: Refactor incrementally. Each phase leaves the codebase in a better state than it was before. You don't need to complete all phases to see benefits -- even Phase 1 alone makes the code more testable and maintainable.

---

## Phase 1: Extract Selectors

Move all inline SOQL into selector classes. This is the **highest-impact, lowest-risk** change because you're not changing any logic -- just moving queries behind a mockable interface.

### What to do

1. **Find all inline SOQL** -- Search for `[SELECT` and `Database.query(` in controllers, services, trigger handlers, and helper classes.
2. **Group by SObject** -- All queries for `Expense_Report__c` go into `ExpenseReportSelector_XX`. One selector per SObject.
3. **Create selector files** -- Interface + class using the `replaceWith` pattern (see Selector template). Move each inline query into a named method like `selectByAccountIds`, `selectPendingByDepartment`.
4. **Replace inline SOQL with selector calls** -- Swap `[SELECT ... FROM X WHERE ...]` with `XSelector_XX.newElevatedInstance().selectByY(ids)`.
5. **Write basic selector tests** -- Simple integration tests with real data for code coverage. These are not logic tests -- just enough to cover the query methods.

### Before → After

```apex
// BEFORE — inline SOQL scattered in a controller
@AuraEnabled
public static List<Expense_Report__c> getReports(Id userId) {
  return [SELECT Id, Name, Status__c, Total_Amount__c
          FROM Expense_Report__c
          WHERE Submitter__c = :userId
          ORDER BY Submitted_Date__c DESC];
}

// AFTER — SOQL moved to selector, controller calls selector
@AuraEnabled
public static List<Expense_Report__c> getReports(Id userId) {
  return ExpenseReportSelector_XX.newElevatedInstance()
    .selectBySubmitterId(new Set<Id>{ userId });
}
```

> **Info:** Result: All SOQL is now centralized, reusable, and mockable. You can already write faster tests by mocking selectors. No business logic has changed.

---

## Phase 2: Extract Services + UnitOfWork

Move business logic out of controllers and trigger handlers into service classes. Move all DML into UnitOfWork.

### What to do

1. **Identify business logic in controllers** -- Any `if` statements, data transformation, validation, or orchestration in controllers belongs in a service.
2. **Create the 3-file service pattern** -- Interface + Facade + Implementation (see Service template). Move all business logic into the impl.
3. **Create UnitOfWork class** -- Define the SObject hierarchy. Replace all direct `insert`/`update`/`delete` in the service impl with `uow.registerNew()`, `uow.registerDirty()`, `uow.registerDeleted()`, and a single `uow.commitWork()`.
4. **Thin out controllers** -- Controllers become try-catch + service call. Nothing else.
5. **Thin out trigger handlers** -- Trigger actions become detect-change + delegate-to-service. Nothing else.

### Before → After

```apex
// BEFORE — controller with business logic + DML
@AuraEnabled
public static void submitReport(Id reportId) {
  Expense_Report__c report = ExpenseReportSelector_XX.newElevatedInstance()
    .selectById(new Set<Id>{ reportId })[0];

  if (report.Status__c != 'Draft') {
    throw new AuraHandledException('Only Draft reports can be submitted.');
  }

  report.Status__c = 'Pending';
  report.Submitted_Date__c = DateTime.now();
  update report;  // Direct DML
}

// AFTER — controller delegates, service has logic + UoW
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
```

> **Info:** Result: Business logic is testable in isolation. You can mock selectors and UoW to write fast unit tests for every service method. Controllers and trigger actions are thin and predictable.

---

## Phase 3: Write Unit Tests (Mocked)

Now that selectors and services are in place, write mocked unit tests for the service layer.

> **Info:** Why now? You need selectors (to mock) and services (to test) before you can write proper unit tests. Phases 1 and 2 make this possible.

### What to do

1. **Write service tests** -- Call the service **facade**. Mock the selector (control the input data) and UoW (verify the output). Test happy paths, error paths, edge cases.
2. **Write controller tests** -- Mock the service. Verify @AuraEnabled wiring and error handling.
3. **Write trigger action tests** -- Build `fflib_TriggerContext` manually. Mock the service. Verify field change detection and correct delegation.
4. **Keep selector tests as coverage-only** -- Simple integration tests with real data. The real selector testing happens through service tests.

> **Info:** Result: Fast, isolated unit tests that verify business logic without touching the database. These run in seconds, not minutes.

---

## Phase 4: Extract Domains

Move in-memory record operations into domain classes. This is the **lowest priority** because the code already works -- domains are a structural improvement, not a functional one.

### When to extract a domain

> **Warning:** Don't create a domain for every SObject. Create one when you have:
> - Multiple places filtering the same list of records by the same criteria
> - Trigger actions that need to filter trigger records before delegating
> - Complex in-memory logic (chained filters, field extraction, record mutation) that makes service methods hard to read

### What to do

1. **Identify repeated in-memory patterns** -- Look for `for` loops that filter lists, extract IDs, or set field values in service impls and trigger actions.
2. **Create domain classes** -- Interface + class extending `fflib_SObjects2`. Move the filtering/extraction/mutation logic into named methods.
3. **Write domain unit tests** -- Pure unit tests with in-memory records. No mocking needed.

### Before → After

```apex
// BEFORE — filtering logic inline in service impl
public void processSubmittedReports(List<Expense_Report__c> reports) {
  Set<Id> pendingSubmitterIds = new Set<Id>();
  for (Expense_Report__c r : reports) {
    if (r.Status__c == 'Pending' && r.Department__c == 'Engineering') {
      pendingSubmitterIds.add(r.Submitter__c);
    }
  }
  // ... use pendingSubmitterIds
}

// AFTER — domain handles the filtering
public void processSubmittedReports(List<Expense_Report__c> reports) {
  IExpenseReports_XX domain = new ExpenseReports_XX(reports);
  Set<Id> pendingSubmitterIds = domain
    .selectByStatus('Pending')
    .selectByDepartment('Engineering')
    .getSubmitterIds();
  // ... use pendingSubmitterIds
}
```

> **Info:** Result: In-memory logic is reusable, chainable, and independently testable. Service methods are shorter and more readable.

---

## Phase 5: Target Test Ratio: 70/30

Once all layers are in place, aim for the following test ratio:

| Ratio | Test Type | What They Cover | Characteristics |
|-------|-----------|-----------------|-----------------|
| **70%** | **Unit tests** (mocked) | Service logic, controller wiring, trigger action delegation, domain filtering | Fast (seconds). No SOQL, no DML. Mock selectors + UoW. Test every branch, edge case, error path. |
| **30%** | **Integration tests** (real data) | End-to-end happy paths, selector coverage, trigger fire-and-forget verification | Slower (seconds to minutes). Insert real data, call real methods, assert real state. Cover the main functionality only. |

> **Info:** What integration tests should cover:
> - Selector query methods (code coverage -- just call it and assert non-empty)
> - 1-2 end-to-end happy paths per feature (e.g., create + submit + approve flow)
> - Trigger actions actually firing (insert a record, verify the trigger action ran)
> - Any code that interacts with platform features you can't mock (approval processes, email, external callouts)

> **Warning:** What integration tests should NOT cover:
> - Every business rule and edge case -- that's what mocked unit tests are for
> - Error paths -- mock the selector to return bad data instead of inserting bad data
> - Permission and sharing -- hard to test reliably, often flaky

---

## Phase Summary

| Phase | What | Unlocks | Risk |
|-------|------|---------|------|
| **1. Selectors** | Move inline SOQL into selector classes | Mockable queries, reusable SOQL, centralized field lists | Low -- no logic change, just moving code |
| **2. Services + UoW** | Extract business logic into services, DML into UoW | Testable business logic, thin controllers, thin triggers | Medium -- logic moves between classes |
| **3. Unit Tests** | Write mocked tests for services, controllers, trigger actions | Fast test suite, high confidence in business logic | None -- additive only |
| **4. Domains** | Extract in-memory record logic into domain classes | Reusable filtering, cleaner service methods | Low -- structural improvement, optional per SObject |
| **5. Test Ratio** | Target 70% mocked unit tests, 30% integration tests | Fast CI, reliable deploys, high coverage | None -- ongoing practice |

> **Tip:** Key takeaway: Start with selectors. They're the lowest risk and unlock everything else. You can't write mocked service tests until you have selectors to mock. You can't thin out controllers until you have services to delegate to. Each phase builds on the one before it.

*Refactoring Roadmap -- April 2026*
