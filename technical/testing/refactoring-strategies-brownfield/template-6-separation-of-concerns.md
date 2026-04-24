# Separation of Concerns

What code goes where, and why. Every layer has one job. When code is in the wrong layer, it becomes hard to test, hard to reuse, and hard to change.

---

## 1. Layer Responsibilities

| Layer | Responsibility | Allowed To | Never Does |
|-------|---------------|------------|------------|
| **Controller** | Entry point for LWC. Resolves user context, logs errors, wraps errors. | Call service facade (static methods). Resolve current user/account. Log errors. | SOQL. DML. Business logic. If statements about data. |
| **Service Facade** | Static API + DI wiring. | One-liner delegates to `newInstance()`. | Any logic at all. It's a pass-through. |
| **Service Impl** | Business logic. Orchestration. | Call selectors. Use domains. Register records to UoW. Validate. Transform data. | Direct SOQL. Direct DML (`insert`, `update`). Direct `Database.query()`. |
| **Selector** | SOQL queries. | `Database.query()`. `newQueryFactory()`. Define default fields. | Business logic. DML. Data transformation. Calling other selectors. |
| **Domain** | In-memory record operations. | Filter records. Extract field values. Mutate field values. | SOQL. DML. Call services. Call selectors. |
| **UnitOfWork** | Transactional DML. | `registerNew`, `registerDirty`, `registerDeleted`, `commitWork`. | Business logic. Queries. |
| **Trigger Action** | Detect trigger changes, delegate. | Read trigger context. Call service or enqueue. Use domain for filtering. | Business logic. Direct SOQL. Direct DML. |

---

## 2. Controller -- Good vs Bad

> **BAD** Controller with business logic and SOQL

```apex
@AuraEnabled
public static void submitExpense(Id reportId) {
  // BAD: SOQL in controller
  Expense_Report__c report = [SELECT Id, Status__c FROM Expense_Report__c WHERE Id = :reportId];

  // BAD: Business logic in controller
  if (report.Status__c != 'Draft') {
    throw new AuraHandledException('Cannot submit');
  }

  // BAD: DML in controller
  report.Status__c = 'Pending';
  update report;
}
```

> **GOOD** Controller delegates to service

```apex
@AuraEnabled
public static void submitExpense(Id reportId) {
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

> **Rule of thumb:** If your controller has more than a try/catch, a logger call, and a single service call, something is in the wrong place.

---

## 3. Service Impl -- Good vs Bad

> **BAD** Service with inline SOQL and DML

```apex
public void submitReport(Id reportId) {
  // BAD: Direct SOQL — should use selector
  Expense_Report__c report = [SELECT Id, Status__c FROM Expense_Report__c WHERE Id = :reportId];

  if (report.Status__c != 'Draft') {
    throw new AuraHandledException('Only Draft reports can be submitted.');
  }

  report.Status__c = 'Pending';

  // BAD: Direct DML — should use UnitOfWork
  update report;
}
```

> **GOOD** Service uses selector + UoW

```apex
public void submitReport(Id reportId) {
  // Selector handles the query
  List<Expense_Report__c> reports = ExpenseReportSelector_XX.newElevatedInstance()
    .selectById(new Set<Id>{ reportId });

  if (reports.isEmpty()) {
    throw new AuraHandledException('Expense report not found.');
  }

  Expense_Report__c report = reports[0];
  if (report.Status__c != 'Draft') {
    throw new AuraHandledException('Only Draft reports can be submitted.');
  }

  // UoW handles the DML
  fflib_ISObjectUnitOfWork uow = UnitOfWork_XX.newInstance();
  report.Status__c = 'Pending';
  uow.registerDirty(report);
  uow.commitWork();
}
```

> **Why this matters:** With inline SOQL/DML, you can't mock anything -- tests need real data, real DML, and are slow. With selectors + UoW, tests are fast, isolated, and verify exact behaviour.

---

## 4. Selector -- Good vs Bad

> **BAD** Selector with business logic

```apex
public List<Expense_Report__c> selectSubmittableByUser(Id userId) {
  List<Expense_Report__c> reports = (List<Expense_Report__c>) Database.query(
    newQueryFactory()
      .setCondition('Submitter__c = :userId')
      .toSOQL()
  );

  // BAD: Business logic in selector — filtering, validation, transformation
  List<Expense_Report__c> result = new List<Expense_Report__c>();
  for (Expense_Report__c r : reports) {
    if (r.Status__c == 'Draft' && r.Total_Amount__c < 10000) {
      r.Status__c = 'Ready';  // BAD: Mutating records in selector
      result.add(r);
    }
  }
  return result;
}
```

> **GOOD** Selector only queries

```apex
// Selector — just the query
public List<Expense_Report__c> selectBySubmitterId(Set<Id> submitterIds) {
  return (List<Expense_Report__c>) Database.query(
    newQueryFactory()
      .setCondition('Submitter__c IN :submitterIds')
      .toSOQL()
  );
}

// Filtering happens in the DOMAIN (in the service impl):
IExpenseReports_XX domain = new ExpenseReports_XX(reports);
List<Expense_Report__c> submittable = domain
  .selectByStatus('Draft')
  .selectUnderBudget(10000)
  .getExpenseReports();
```

---

## 5. Domain -- Good vs Bad

> **BAD** Domain doing SOQL or calling services

```apex
public IExpenseReports_XX enrichWithApproverNames() {
  // BAD: SOQL in domain
  Set<Id> approverIds = getApproverIds();
  Map<Id, User> approvers = new Map<Id, User>(
    [SELECT Id, Name FROM User WHERE Id IN :approverIds]
  );

  // BAD: Calling a service from domain
  NotificationService_XX.notifyApprovers(approverIds);

  return this;
}
```

> **GOOD** Domain only operates on in-memory records

```apex
// Domain — extract IDs for the service to use
public Set<Id> getApproverIds() {
  Set<Id> result = new Set<Id>();
  for (Expense_Report__c record : getExpenseReports()) {
    if (record.Approver__c != null) {
      result.add(record.Approver__c);
    }
  }
  return result;
}

// The SERVICE orchestrates:
Set<Id> approverIds = domain.getApproverIds();
Map<Id, User> approvers = UserSelector_XX.newElevatedInstance()
  .selectByIds(approverIds);
NotificationService_XX.notifyApprovers(approverIds);
```

---

## 6. Trigger Action -- Good vs Bad

> **BAD** Trigger action with business logic, SOQL, and DML

```apex
public override void onAfterUpdate() {
  for (Expense_Report__c record : (List<Expense_Report__c>) triggerContext.getRecords()) {
    Expense_Report__c old = (Expense_Report__c) triggerContext.existingRecords.get(record.Id);
    if (old.Status__c != record.Status__c && record.Status__c == 'Pending') {
      // BAD: SOQL in trigger action
      User approver = [SELECT Email FROM User WHERE Id = :record.Approver__c];
      // BAD: Business logic in trigger action
      Messaging.SingleEmailMessage mail = new Messaging.SingleEmailMessage();
      mail.setToAddresses(new List<String>{ approver.Email });
      mail.setSubject('Expense report needs approval');
      // BAD: DML in trigger action
      Messaging.sendEmail(new List<Messaging.SingleEmailMessage>{ mail });
    }
  }
}
```

> **GOOD** Trigger action detects change, delegates to service

```apex
public override void onAfterUpdate() {
  Set<Id> changedIds = new Map<Id, SObject>(
    triggerContext.getChangedRecords(Expense_Report__c.Status__c)
  ).keySet();

  if (!changedIds.isEmpty()) {
    NotificationService_XX.notifyApprovers(changedIds);
  }
}
```

> **Trigger action rule:** Detect what changed. Delegate. That's it. If you're writing a `for` loop with business decisions inside a trigger action, that logic belongs in a service.

---

## 7. Quick Decision Guide

| I need to... | Put it in... |
|--------------|-------------|
| Query records from the database | **Selector** |
| Filter records already in memory | **Domain** |
| Extract IDs or field values from a list of records | **Domain** |
| Set field values on a list of records | **Domain** (mutator) |
| Validate a business rule | **Service Impl** |
| Decide what happens based on data | **Service Impl** |
| Insert, update, or delete records | **UnitOfWork** (called from Service Impl) |
| Respond to a trigger event | **Trigger Action** (detect + delegate to Service) |
| Expose a method to LWC | **Controller** (@AuraEnabled, delegates to Service) |
| Send an email / enqueue a job | **Service Impl** |
| Transform JSON input into SObjects | **Service Impl** |
| Call an external API | **Service Impl** (or a dedicated integration service) |

---

## 8. The Golden Rules

| # | Rule |
|---|------|
| 1 | **Controllers are thin.** Try-catch + service call. Nothing else. |
| 2 | **Service facades have zero logic.** One-liner static delegates only. |
| 3 | **All business logic lives in Service Impl.** Validation, orchestration, transformation. |
| 4 | **Selectors only query.** No filtering, no mutation, no business decisions. |
| 5 | **Domains never touch the database.** No SOQL, no DML, no service calls. |
| 6 | **All DML goes through UnitOfWork.** Never `insert`/`update`/`delete` directly. |
| 7 | **Trigger actions detect and delegate.** No business logic, no SOQL, no DML. |
| 8 | **Each layer depends only on the layer below it.** Controllers -> Services -> Selectors/Domains/UoW. Never backwards. |

---

*Separation of Concerns Guide -- April 2026*
