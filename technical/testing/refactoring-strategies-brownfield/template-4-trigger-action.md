# Trigger Action Template

Trigger actions are **small, focused classes** that respond to a single trigger event on a single SObject. They are bound to their SObject and context via **Custom Metadata**.

> **Info:** How it works:
>
> 1. A trigger fires on an SObject
> 2. The `fflib_TriggerHandler` reads Custom Metadata to find which trigger action classes to run
> 3. It creates an instance, passes the trigger context, and calls `doWork()`
> 4. Your class receives the trigger records and decides what to do

> **Warning:** Files to create:
>
> 1. `TA_ExpenseReportNotify_XX.cls` -- Trigger Action class
> 2. `TA_ExpenseReportNotifyTest_XX.cls` -- Test class
> 3. `fflib_TriggerAction.ExpenseReportNotify_XX.md-meta.xml` -- Custom Metadata binding
>
> **Naming:** `TA_{SObject}{WhatItDoes}_{Suffix}`

---

## 1. Trigger Action Examples

### A. After Update -- Detect field changes and delegate to service

The most common pattern. Detects which records had a specific field change and passes them to a service.

```apex
public inherited sharing class TA_ExpenseReportNotify_XX extends fflib_TriggerAction {

  public override void onAfterUpdate() {
    notifyApproversOnSubmit();
  }

  private void notifyApproversOnSubmit() {
    // getChangedRecords() returns ONLY records where Status__c actually changed
    Set<Id> changedIds = new Map<Id, SObject>(
      triggerContext.getChangedRecords(Expense_Report__c.Status__c)
    ).keySet();

    if (!changedIds.isEmpty()) {
      // Delegate to the service layer — trigger action stays thin
      NotificationService_XX.notifyApprovers(changedIds);
    }
  }
}
```

### B. Before Insert -- Set default field values

Runs before the record is saved. Modify the records directly (no UoW needed for before triggers).

```apex
public inherited sharing class TA_ExpenseReportDefaults_XX extends fflib_TriggerAction {

  public override void onBeforeInsert() {
    applyDefaults();
  }

  private void applyDefaults() {
    for (Expense_Report__c record :
        (List<Expense_Report__c>) triggerContext.getRecords()) {
      if (String.isBlank(record.Status__c)) {
        record.Status__c = 'Draft';
      }
      if (record.Submitter__c == null) {
        record.Submitter__c = UserInfo.getUserId();
      }
    }
  }
}
```

### C. After Update -- Use a Domain for in-memory filtering

When you need to filter the changed records before acting on them.

```apex
public inherited sharing class TA_ExpenseReportAutoApprove_XX extends fflib_TriggerAction {

  private static final Decimal AUTO_APPROVE_THRESHOLD = 100.00;

  public override void onAfterUpdate() {
    autoApproveSmallExpenses();
  }

  private void autoApproveSmallExpenses() {
    List<Expense_Report__c> submitted = (List<Expense_Report__c>)
      triggerContext.getChangedRecords(Expense_Report__c.Status__c);

    if (submitted.isEmpty()) {
      return;
    }

    // Use domain to filter in memory — no extra SOQL
    IExpenseReports_XX domain = new ExpenseReports_XX(submitted);
    Set<Id> smallExpenseIds = domain
      .selectByStatus('Pending')
      .selectUnderBudget(AUTO_APPROVE_THRESHOLD)
      .getIds();

    if (!smallExpenseIds.isEmpty()) {
      ExpenseService_XX.autoApprove(smallExpenseIds);
    }
  }
}
```

---

## 2. Custom Metadata Record

This XML file tells the trigger handler: "run `TA_ExpenseReportNotify_XX` after updates on `Expense_Report__c`".

```xml
<?xml version="1.0" encoding="UTF-8"?>
<CustomMetadata xmlns="http://soap.sforce.com/2006/04/metadata"
                xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
                xmlns:xsd="http://www.w3.org/2001/XMLSchema">
    <label>XX_ExpenseReport_Notify</label>
    <protected>false</protected>
    <values>
        <field>AfterDelete__c</field>
        <value xsi:type="xsd:boolean">false</value>
    </values>
    <values>
        <field>AfterInsert__c</field>
        <value xsi:type="xsd:boolean">false</value>
    </values>
    <values>
        <field>AfterUndelete__c</field>
        <value xsi:type="xsd:boolean">false</value>
    </values>
    <values>
        <field>AfterUpdate__c</field>
        <value xsi:type="xsd:boolean">true</value>
    </values>
    <values>
        <field>BeforeDelete__c</field>
        <value xsi:type="xsd:boolean">false</value>
    </values>
    <values>
        <field>BeforeInsert__c</field>
        <value xsi:type="xsd:boolean">false</value>
    </values>
    <values>
        <field>BeforeUpdate__c</field>
        <value xsi:type="xsd:boolean">false</value>
    </values>
    <values>
        <field>ExecutionContext__c</field>
        <value xsi:type="xsd:string">Realtime</value>
    </values>
    <values>
        <field>ImplementationType__c</field>
        <value xsi:type="xsd:string">TA_ExpenseReportNotify_XX</value>
    </values>
    <values>
        <field>ObjectTypeAlternate__c</field>
        <value xsi:nil="true"/>
    </values>
    <values>
        <field>ObjectType__c</field>
        <value xsi:type="xsd:string">Expense_Report__c</value>
    </values>
    <values>
        <field>Sequence__c</field>
        <value xsi:type="xsd:double">1.0</value>
    </values>
    <values>
        <field>Stateful__c</field>
        <value xsi:type="xsd:boolean">false</value>
    </values>
</CustomMetadata>
```

> **Tip:** `Sequence__c` controls execution order. If two trigger actions run on the same SObject and context, the one with the lower sequence runs first.

---

## 3. Unit Test

Trigger action tests build a `fflib_TriggerContext` manually -- **no actual DML or trigger execution**. This makes them fast and isolated.

```apex
@IsTest(IsParallel=true)
private class TA_ExpenseReportNotifyTest_XX {

  @IsTest
  static void itShouldNotifyWhenStatusChanges() {
    // given ─────────────────────────────────────────────────
    Id reportId = fflib_IDGenerator.generate(Expense_Report__c.SObjectType);

    // Old record (before the update)
    Expense_Report__c oldRecord = new Expense_Report__c(
      Id = reportId, Status__c = 'Draft'
    );

    // New record (after the update)
    Expense_Report__c newRecord = new Expense_Report__c(
      Id = reportId, Status__c = 'Pending'
    );

    // Build trigger context — simulates Trigger.new and Trigger.oldMap
    fflib_TriggerContext ctx = new fflib_TriggerContext(
      new List<Expense_Report__c>{ newRecord }
    );
    ctx.existingRecords = new Map<Id, SObject>{ reportId => oldRecord };
    ctx.triggerOperation = System.TriggerOperation.AFTER_UPDATE;

    // Mock the service the trigger action delegates to
    fflib_ApexMocks mocks = new fflib_ApexMocks();
    INotificationService_XX mockService =
      (INotificationService_XX) mocks.mock(INotificationService_XX.class);
    NotificationService_XX.replaceWith(mockService);

    // Create and wire the trigger action
    fflib_ITriggerAction triggerAction = new TA_ExpenseReportNotify_XX();
    triggerAction.setContext(ctx);

    // when ──────────────────────────────────────────────────
    Test.startTest();
    triggerAction.doWork();
    Test.stopTest();

    // then ──────────────────────────────────────────────────
    ((INotificationService_XX) mocks.verify(mockService, 1))
      .notifyApprovers(new Set<Id>{ reportId });
  }

  @IsTest
  static void itShouldNotNotifyWhenStatusUnchanged() {
    // given
    Id reportId = fflib_IDGenerator.generate(Expense_Report__c.SObjectType);

    Expense_Report__c oldRecord = new Expense_Report__c(
      Id = reportId, Status__c = 'Draft'
    );
    Expense_Report__c newRecord = new Expense_Report__c(
      Id = reportId, Status__c = 'Draft'  // Same — no change
    );

    fflib_TriggerContext ctx = new fflib_TriggerContext(
      new List<Expense_Report__c>{ newRecord }
    );
    ctx.existingRecords = new Map<Id, SObject>{ reportId => oldRecord };
    ctx.triggerOperation = System.TriggerOperation.AFTER_UPDATE;

    fflib_ApexMocks mocks = new fflib_ApexMocks();
    INotificationService_XX mockService =
      (INotificationService_XX) mocks.mock(INotificationService_XX.class);
    NotificationService_XX.replaceWith(mockService);

    fflib_ITriggerAction triggerAction = new TA_ExpenseReportNotify_XX();
    triggerAction.setContext(ctx);

    // when
    Test.startTest();
    triggerAction.doWork();
    Test.stopTest();

    // then — service should NOT have been called
    ((INotificationService_XX) mocks.verify(mockService, 0))
      .notifyApprovers((Set<Id>) fflib_Match.anyObject());
  }
}
```

---

## Quick Reference

| What | Pattern |
| --- | --- |
| Parent class | `fflib_TriggerAction` |
| Override methods | `onBeforeInsert`, `onBeforeUpdate`, `onAfterInsert`, `onAfterUpdate`, `onBeforeDelete`, `onAfterDelete`, `onAfterUnDelete` |
| Get all trigger records | `triggerContext.getRecords()` |
| Get changed records only | `triggerContext.getChangedRecords(SObjectField)` |
| Get old record values | `triggerContext.existingRecords` |
| Binding | **Custom Metadata** (`fflib_TriggerAction__mdt`) -- trigger actions still use CMT |
| Test pattern | Build `fflib_TriggerContext` manually -> `.setContext(ctx)` -> `.doWork()` |
| Naming | `TA_{SObject}{Action}_{Suffix}` |
| Keep it thin | Detect changes, delegate to service. No business logic in trigger actions. |

---

*Trigger Action Template -- April 2026*
