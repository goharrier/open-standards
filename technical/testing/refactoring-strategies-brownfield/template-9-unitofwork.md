# UnitOfWork Template

The UnitOfWork (UoW) collects all DML operations (insert, update, delete) and executes them in a **single transaction** at the end. Instead of writing `insert record;` or `update record;` scattered throughout your code, you register records with the UoW and call `commitWork()` once.

---

## 1. Why Does UoW Exist?

> **BAD** -- DML scattered everywhere

```apex
public void processExpenses(List<Expense_Report__c> reports) {
  // Three separate DML operations = three savepoints, three sets of triggers
  insert reports;

  List<Expense_Line__c> lines = buildLines(reports);
  insert lines;

  List<Task> tasks = buildFollowUpTasks(reports);
  insert tasks;
  // If the third insert fails, the first two already committed.
  // You now have orphaned data.
}
```

> **GOOD** -- All DML in one transaction

```apex
public void processExpenses(List<Expense_Report__c> reports) {
  fflib_ISObjectUnitOfWork uow = UnitOfWork_XX.newInstance();

  for (Expense_Report__c report : reports) {
    uow.registerNew(report);

    for (Expense_Line__c line : buildLines(report)) {
      uow.registerNew(line, Expense_Line__c.Expense_Report__c, report);
    }
  }

  // One commit = one transaction. If anything fails, everything rolls back.
  uow.commitWork();
}
```

> **Benefits of UoW:**
>
> - **Transactional safety** -- all-or-nothing. If any DML fails, everything rolls back.
> - **Parent-child linking** -- register a child with a parent that hasn't been inserted yet. UoW resolves the IDs after insert.
> - **Fewer DML statements** -- UoW batches all inserts of the same SObjectType into a single `insert` call. Helps with governor limits.
> - **Fully mockable** -- in tests, replace UoW with a mock to verify exactly which records were registered, without any real DML.

---

## 2. Why Its Own Class (Not Part of Application)

> **Old pattern:** The UoW factory lived inside the `Application` class alongside selectors and services:
> `Application_PP.UnitOfWork.newInstance()`
>
> **New pattern:** UoW gets its own standalone class:
> `UnitOfWork_PP.newInstance()`

Why move it out?

| Reason | Explanation |
|--------|-------------|
| **Consistency with selectors & services** | Selectors and services already use `replaceWith` on their own class. UoW should follow the same pattern. One pattern to learn, not two. |
| **The Application class becomes unnecessary** | Selectors use `MySelector.replaceWith()`, services use `MyService.replaceWith()`, UoW uses `UnitOfWork_XX.replaceWith()`, and domains are instantiated directly with `new`. Every component manages its own DI -- the Application class has nothing left to do. |
| **Different modules need different hierarchies** | Partner Portal works with `Pre_Approval__c` and `PA_Activity__c`. Customer Success works with `Account` and `AccountTeamMember`. Each module defines its own UoW with only the SObjects it needs. A shared Application class can't easily handle this. |
| **Elevated DML** | The UoW class is the right place to define an `ElevatedDml` inner class that runs DML in SYSTEM_MODE. This belongs next to the factory, not buried in a generic Application class. |
| **Testability** | `UnitOfWork_XX.replaceWith(mockUow)` is direct and obvious. No need to remember which Application class to call `setMock` on. |

---

## 3. UnitOfWork Class Template

> **File to create:**
> 1. `UnitOfWork_XX.cls` -- one per module/package

```apex
public with sharing class UnitOfWork_XX {

  // ── SObject Hierarchy ────────────────────────────────────
  // Listed in PARENT-TO-CHILD order.
  // UoW inserts top-down and deletes bottom-up, so parents
  // exist before children reference them.

  public final static List<SObjectType> UNIT_OF_WORK_HIERARCHY {
    get {
      if (UNIT_OF_WORK_HIERARCHY == null) {
        UNIT_OF_WORK_HIERARCHY = new List<SObjectType>{
          Expense_Report__c.SObjectType,    // parent
          Expense_Line__c.SObjectType,      // child of Expense_Report__c
          ContentVersion.SObjectType,       // attachment
          ContentDocumentLink.SObjectType   // links attachment to parent
        };
      }
      return UNIT_OF_WORK_HIERARCHY;
    }
    private set;
  }

  // ── DI Wiring ────────────────────────────────────────────
  // Two replacement options:
  // - instanceReplacement: replaces the entire UoW (for mocking in tests)
  // - dmlReplacement: replaces just the DML layer (for elevated access)

  private static fflib_SObjectUnitOfWork.IDML dmlReplacement;
  private static fflib_ISObjectUnitOfWork instanceReplacement;

  // ── Factory Methods ──────────────────────────────────────

  public static fflib_ISObjectUnitOfWork newInstance() {
    if (instanceReplacement != null) {
      return instanceReplacement;
    }
    return (dmlReplacement == null)
      ? new fflib_SObjectUnitOfWork(UNIT_OF_WORK_HIERARCHY)
      : new fflib_SObjectUnitOfWork(UNIT_OF_WORK_HIERARCHY, dmlReplacement);
  }

  public static fflib_ISObjectUnitOfWork newInstance(fflib_SObjectUnitOfWork.IDML dml) {
    if (instanceReplacement != null) {
      return instanceReplacement;
    }
    fflib_SObjectUnitOfWork.IDML effectiveDml = (dmlReplacement != null) ? dmlReplacement : dml;
    return new fflib_SObjectUnitOfWork(UNIT_OF_WORK_HIERARCHY, effectiveDml);
  }

  public static fflib_ISObjectUnitOfWork newInstance(List<SObjectType> hierarchy) {
    if (instanceReplacement != null) {
      return instanceReplacement;
    }
    return (dmlReplacement == null)
      ? new fflib_SObjectUnitOfWork(hierarchy)
      : new fflib_SObjectUnitOfWork(hierarchy, dmlReplacement);
  }

  // ── Test Replacement ─────────────────────────────────────

  // Replace the entire UoW instance (for mocking all DML in tests)
  @TestVisible
  private static void replaceWith(fflib_ISObjectUnitOfWork replacement) {
    UnitOfWork_XX.instanceReplacement = replacement;
  }

  // Replace just the DML implementation (for elevated access or test DML mocking)
  @TestVisible
  private static void replaceDmlWith(fflib_SObjectUnitOfWork.IDML replacement) {
    UnitOfWork_XX.dmlReplacement = replacement;
  }

  // ── Elevated DML ─────────────────────────────────────────
  // Use when you need DML in SYSTEM_MODE (bypasses sharing/FLS).
  // Usage: UnitOfWork_XX.newInstance(new UnitOfWork_XX.ElevatedDml())

  public without sharing class ElevatedDml extends fflib_SObjectUnitOfWork.SimpleDML {
    @TestVisible
    private AccessLevel m_accessLevel;

    public ElevatedDml() {
      this(AccessLevel.SYSTEM_MODE);
    }

    public ElevatedDml(AccessLevel access) {
      m_accessLevel = access;
    }

    public virtual override void dmlInsert(List<SObject> objList) {
      Database.insert(objList, m_accessLevel);
    }

    public virtual override void dmlUpdate(List<SObject> objList) {
      Database.update(objList, m_accessLevel);
    }

    public virtual override void dmlDelete(List<SObject> objList) {
      Database.delete(objList, m_accessLevel);
    }
  }
}
```

---

## 4. The Hierarchy -- Why Order Matters

The `UNIT_OF_WORK_HIERARCHY` list must be in **parent-to-child** order. UoW processes operations in this order:

| Operation | Order | Why |
|-----------|-------|-----|
| **Insert** | Top -> Bottom | Parents are inserted first so child records can reference the parent's new Id. |
| **Update** | Top -> Bottom | Parents updated before children, in case children depend on parent field values. |
| **Delete** | Bottom -> Top | Children deleted first so there are no orphaned references when the parent is deleted. |

> **Common mistake:** If you register a child record linked to a parent but the parent's SObjectType isn't listed *before* the child in the hierarchy, UoW will throw an error at `commitWork()`. Always add new SObjectTypes to the hierarchy in the correct position.

---

## 5. How to Use in Service Impl

### A. Basic -- insert, update, delete

```apex
fflib_ISObjectUnitOfWork uow = UnitOfWork_XX.newInstance();

// Insert a new record
uow.registerNew(new Expense_Report__c(
  Submitter__c = userId,
  Status__c = 'Draft'
));

// Update an existing record
report.Status__c = 'Submitted';
uow.registerDirty(report);

// Delete a record
uow.registerDeleted(oldLine);

// One commit — all three operations in one transaction
uow.commitWork();
```

### B. Parent-child linking (parent not yet inserted)

```apex
fflib_ISObjectUnitOfWork uow = UnitOfWork_XX.newInstance();

// Parent — doesn't have an Id yet
Expense_Report__c report = new Expense_Report__c(Status__c = 'Draft');
uow.registerNew(report);

// Child — linked to parent via relationship field
// UoW will set Expense_Report__c after the parent is inserted
Expense_Line__c line = new Expense_Line__c(Description__c = 'Flight', Amount__c = 450);
uow.registerNew(line, Expense_Line__c.Expense_Report__c, report);

uow.commitWork();
// After commit: line.Expense_Report__c == report.Id (auto-resolved)
```

### C. Elevated DML (bypass sharing)

```apex
// Use when the service needs to write records the current user
// doesn't have sharing access to
fflib_ISObjectUnitOfWork uow = UnitOfWork_XX.newInstance(
  new UnitOfWork_XX.ElevatedDml()
);

uow.registerNew(sensitiveRecord);
uow.commitWork();  // Runs in SYSTEM_MODE
```

---

## 6. How to Mock in Tests

### A. Mock the full instance (most common)

Replaces the entire UoW so no DML happens. You then **verify** which records were registered.

```apex
// given
fflib_ApexMocks mocks = new fflib_ApexMocks();
fflib_ISObjectUnitOfWork mockUow =
  (fflib_ISObjectUnitOfWork) mocks.mock(fflib_ISObjectUnitOfWork.class);

// Inject the mock — all UnitOfWork_XX.newInstance() calls return this mock
UnitOfWork_XX.replaceWith(mockUow);

// when
Test.startTest();
ExpenseService_XX.createReport(userId, data, lines);
Test.stopTest();

// then — verify what was registered (no real DML happened)
((fflib_ISObjectUnitOfWork) mocks.verify(mockUow, 1))
  .registerNew(
    (Expense_Report__c) fflib_Match.sObjectWith(
      new Map<Schema.SObjectField, Object>{
        Expense_Report__c.Status__c => 'Draft'
      }
    )
  );
((fflib_ISObjectUnitOfWork) mocks.verify(mockUow, 1)).commitWork();
```

### B. replaceWith vs replaceDmlWith

| Method | What it does | When to use |
|--------|-------------|-------------|
| `replaceWith(mockUow)` | Replaces the entire UoW instance. `newInstance()` returns the mock directly. | **Most tests.** You want to verify `registerNew` / `registerDirty` / `commitWork` calls without any real DML. |
| `replaceDmlWith(mockDml)` | Replaces only the DML layer inside a real UoW. The UoW still tracks records normally, but `dmlInsert` / `dmlUpdate` / `dmlDelete` call the mock. | **Rare.** When you need the UoW to actually track and resolve parent-child relationships, but don't want real DML to fire. |

---

## 7. One UoW Per Module

Each module/package gets its own UoW class because each module works with different SObjects:

| Module | UoW Class | Hierarchy |
|--------|-----------|-----------|
| Partner Portal | `UnitOfWork_PP` | `Partner_Application_Form__c, PartnerPurchasePathPreference__c, ContentVersion, ContentDocumentLink` |
| Customer Success | `UnitOfWork_CS` | `Account, AccountTeamMember` |
| Your New Module | `UnitOfWork_XX` | Your module's SObjects in parent-to-child order |

> **Why not one giant shared UoW?** A single hierarchy containing every SObject in the org would be massive, hard to maintain, and would create unnecessary coupling between modules. Each module knows which SObjects it touches -- its UoW should reflect only those.

---

## Quick Reference

| What | Pattern |
|------|---------|
| Class per module | `UnitOfWork_PP`, `UnitOfWork_CS`, `UnitOfWork_XX` |
| Hierarchy order | Parent -> Child (inserts top-down, deletes bottom-up) |
| Create in service impl | `fflib_ISObjectUnitOfWork uow = UnitOfWork_XX.newInstance()` |
| Elevated DML | `UnitOfWork_XX.newInstance(new UnitOfWork_XX.ElevatedDml())` |
| Register insert | `uow.registerNew(record)` |
| Register insert with parent | `uow.registerNew(child, Child.ParentField, parent)` |
| Register update | `uow.registerDirty(record)` |
| Register delete | `uow.registerDeleted(record)` |
| Execute all DML | `uow.commitWork()` |
| Mock (full instance) | `UnitOfWork_XX.replaceWith(mockUow)` |
| Mock (DML only) | `UnitOfWork_XX.replaceDmlWith(mockDml)` |
| Application class | **Not needed** |
| Custom Metadata | **Not needed** |
