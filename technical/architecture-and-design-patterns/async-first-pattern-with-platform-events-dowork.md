# Async Processing with Platform Events (DoWork)

## **Overview**

{% hint style="info" %}
**When to use this pattern:** When async work requires **callouts to external systems**, **multiple independent subscribers**, **cross-org communication**, or **CometD/Empapi streaming to clients**. For simple async chaining within Apex (record updates, calculations, sequential jobs), prefer the [Queueable Chain Manager](queueable-chain-manager.md) pattern instead.
{% endhint %}

#### **Purpose**

The DoWork pattern uses Platform Events as a lightweight message bus for asynchronous execution. Work items serialize themselves, publish as events, and a trigger deserializes and executes them in a separate transaction with automatic retry capabilities.

#### **When Platform Events Are the Right Choice**

Platform Events add real value over Queueable Apex in specific scenarios:

| Scenario | Why Platform Events | Why Not Queueable |
| --- | --- | --- |
| **External callouts** | Separate transaction; callout failures don't roll back DML | Callouts in Queueable work too, but PE gives you built-in retry and decoupling |
| **Multiple subscriber types** | Apex trigger, Flows, CometD clients, and Empapi can all subscribe to the same event | Queueable is single-consumer by nature |
| **Cross-org / external consumers** | CometD, Empapi, and external systems can subscribe | Queueable is internal-only |
| **Fan-out processing** | Publish once, many subscriber types react independently | Queueable chains are sequential |
| **Publish before commit** | With "Publish Immediately" mode, events publish before transaction commit | Queueable enqueues at commit |
| **Fire-and-forget from LWC/Flow** | Platform Events are declaratively accessible | Queueable requires Apex invocation |

#### **When NOT to Use Platform Events**

* **Simple async record updates** → Use [Queueable Chain Manager](queueable-chain-manager.md)
* **Sequential job chaining** → Use [Queueable Chain Manager](queueable-chain-manager.md)
* **Operations needing return values** → Use Queueable with polling or Platform Event callback
* **Low-volume, single-consumer async** → Queueable is simpler and has no payload serialization overhead
* **Operations that must respect transaction rollback** → With "Publish Immediately" mode, events cannot be rolled back (use "Publish After Commit" if rollback safety is needed, but then you lose the immediate decoupling benefit)

### **Problem Statement**

#### **The Challenge**

Certain async operations need capabilities that Queueable Apex cannot provide:

1. **Callout Decoupling**: External system calls should not be coupled to the triggering DML transaction
2. **Fan-Out**: A single business event needs to trigger multiple independent reactions
3. **External Subscribers**: External systems or LWC components need to react to server-side events
4. **Retry Semantics**: Transient failures (e.g., external API timeouts) need automatic retry without custom orchestration

### **Solution**

#### **Core Concept**

Work items implement a common interface, serialize to JSON, publish as a Platform Event, and are deserialized and executed by a trigger handler in a separate transaction. The retry count travels with the event.

#### **1. Define the Work Interface**

```apex
public interface IDoWork {
    /**
     * Executes all the work of the work item
     */
    void doWork();

    /**
     * @return Returns the name of the work item, used for logging purposes
     */
    String getClassName();

    /**
     * When the maximum retries has been reached this method is invoked
     * @param e The last thrown exception
     */
    void onException(Exception e);

    /**
     * Logic to execute on finally (after possible exception handling)
     */
    void onFinally();

    void publish();
    void publish(Integer retries);
}
```

#### **2. Create the Abstract Base Class**

The abstract class handles serialization and Platform Event publication:

```apex
public abstract class DoWorkAbstract implements IDoWork {

    public virtual void publish() {
        publish(0);
    }

    /**
     * @param retries The number of retries
     */
    public virtual void publish(Integer retries) {
        String serialized = JSON.serialize(this);
        if (serialized.length() > 131072) {
            throw new WorkException('Work item is too large');
        }

        Database.SaveResult results = EventBus.publish(
            new DoWork__e(
                Work__c = serialized,
                ClassName__c = getClassName(),
                Retries__c = retries
            )
        );

        if (results.isSuccess() == false) {
            String errorMessage = 'Unable to publish work item: ';
            for (Database.Error err : results.getErrors()) {
                errorMessage += err.getStatusCode() + ' - ' + err.getMessage();
            }
            throw new WorkException(errorMessage);
        }
    }

    public virtual void onException(Exception e) {
        // Override with your own logic if required
    }

    public virtual void onFinally() {
        // Override with your own logic if required
    }

    abstract public String getClassName();

    public class WorkException extends Exception {}
}
```

#### **3. Platform Event Trigger Handler**

The trigger deserializes and executes work items with retry logic:

```apex
trigger DoWorkTrigger on DoWork__e (after insert) {
    for (DoWork__e workItem : Trigger.new) {
        if (String.isBlank(workItem.Work__c)) continue;

        Type className = System.Type.forName(workItem.ClassName__c);
        if (className == null) {
            Logger.error('Workitem has an incorrect class name: ' + workItem.ClassName__c, workItem);
            continue;
        }

        IDoWork work;
        try {
            work = (IDoWork) JSON.deserialize(workItem.Work__c, className);
        } catch (Exception e) {
            Logger.error('Invalid JSON for work item #' + workItem.Id + ': ' + workItem.Work__c);
            continue;
        }

        try {
            work.doWork();
        } catch (Exception e) {
            if (workItem.Retries__c > 0) {
                work.publish(Integer.valueOf(workItem.Retries__c) - 1);
                Logger.info('Retrying work item ' + work.getClassName() + ' due to: ' + e.getMessage());
            } else {
                Logger.error('Failing to execute work item ' + work.getClassName() + ' due to: ' + e.getMessage());
                work.onException(e);
            }
        } finally {
            try {
                work.onFinally();
            } catch (Exception e) {
                // Ignore finally error
            }
        }
    }
    Logger.saveLog();
}
```

#### **Key Components**

| Component | Purpose | Responsibility |
| --- | --- | --- |
| `IDoWork` | Interface | Defines the contract for async work items |
| `DoWorkAbstract` | Abstract Base | Handles serialization and event publication |
| `DoWork__e` | Platform Event | Carries serialized work data between transactions |
| `DoWorkTrigger` | Event Handler | Deserializes and executes work items with retry |

### **Implementation Details**

#### **Required Setup**

1. **Create Platform Event**: `DoWork__e` with fields:
   * `Work__c` (Long Text Area, 131072 chars) — Serialized work item
   * `ClassName__c` (Text, 255) — Fully qualified class name
   * `Retries__c` (Number) — Remaining retry attempts
2. **Deploy Abstract Classes**: `IDoWork` interface and `DoWorkAbstract` class
3. **Create Trigger**: `DoWorkTrigger` on `DoWork__e`

#### **Concrete Worker Example — External Callout**

This is the ideal use case for the DoWork pattern: an external system callout triggered by a record change.

```apex
/**
 * Syncs account data to an external ERP system after creation.
 * Uses Platform Events because:
 * 1. Callout failure should not roll back the Account insert
 * 2. Built-in retry handles transient API failures
 * 3. External monitoring tools subscribe to DoWork__e for observability
 */
public without sharing class AsyncErpAccountSync extends DoWorkAbstract {

    private final Set<Id> accountIds;

    public AsyncErpAccountSync(Set<Id> accountIds) {
        this.accountIds = accountIds;
    }

    public void doWork() {
        List<Account> accounts = [
            SELECT Name, AccountNumber, BillingAddress, Industry
            FROM Account
            WHERE Id IN :accountIds
        ];

        ErpIntegrationService.syncAccounts(accounts);
    }

    public override void onException(Exception e) {
        // Notify integration team when retries exhausted
        IntegrationAlertService.notifyFailure('ERP Account Sync', accountIds, e);
    }

    public override String getClassName() {
        return 'AsyncErpAccountSync';
    }
}
```

#### **Publishing with Retries**

For callouts to external systems that may have transient failures:

```apex
// Allow 3 retries for transient API failures
new AsyncErpAccountSync(accountIds).publish(3);
```

#### **Preventing Duplicate Scheduling**

When working with triggers, prevent the same async job from being scheduled multiple times in the same transaction:

```apex
public with sharing class AccountTriggerHandler extends fflib_SObjectDomain {

    private static Set<String> scheduledAsyncTasks = new Set<String>();

    private static void syncToErp(IAccounts accounts) {
        if (scheduledAsyncTasks.contains('AsyncErpAccountSync')) return;

        new AsyncErpAccountSync(accounts.getRecordIds()).publish(3);
        scheduledAsyncTasks.add('AsyncErpAccountSync');
    }
}
```

### **Best Practices**

#### **Do's**

* Use for callouts, fan-out, and multi-subscriber scenarios
* Implement `onException` for critical workflows that need failure notification
* Use retries for transient failures (callout timeouts, lock contention)
* Keep serialized payloads small — pass record IDs and query fresh data in `doWork()`
* Include meaningful class names for debugging and monitoring
* Use static tracking sets to prevent duplicate scheduling within transactions

#### **Don'ts**

* Do not use for simple async record updates — use [Queueable Chain Manager](queueable-chain-manager.md) instead
* Do not serialize large object graphs (query fresh data in `doWork()`)
* Do not rely on execution order — Platform Events may be processed out of order
* Do not store sensitive data in the event payload (it's visible in Event Monitoring)
* Do not use when you need transaction rollback semantics with "Publish Immediately" mode — events fire regardless of commit outcome

### **Considerations**

#### **Publish Behavior**

Platform Events support two publish modes, configured per event definition:

* **Publish After Commit** (default since Summer '18) — Events are published only after the transaction commits successfully. If the transaction rolls back, events are discarded. Use this for most scenarios.
* **Publish Immediately** — Events publish before commit and cannot be rolled back. Use this when you need the subscriber to act immediately (e.g., the DoWork pattern where decoupling from the originating transaction is the whole point).

{% hint style="warning" %}
Our `DoWork__e` should use **Publish Immediately** to ensure work items fire even if the originating transaction has complex DML. If you switch to "Publish After Commit", be aware that a transaction rollback will silently discard the work item.
{% endhint %}

#### **Governor Limits**

* Platform Events have their own limits separate from the triggering transaction
* Event payload is limited to 1MB total, but individual Long Text fields max at 131,072 characters
* Event allocations are measured as **maximum deliveries per hour** and vary by org edition — check Setup → Platform Events → Usage to see your org's limits
* Only one Apex trigger is allowed per Platform Event object (same as standard sObjects), but multiple subscriber types (Apex trigger, Flow, CometD, Empapi) can listen independently

#### **Performance Impact**

* With "Publish Immediately", events fire before transaction commit; with "Publish After Commit", they fire after
* Parallel processing of events provides horizontal scalability
* Each event fires in its own transaction with full governor limits

#### **Security Implications**

* Event payload is stored temporarily and visible in Event Monitoring
* Use appropriate sharing settings (`with sharing` vs `without sharing`) in work classes
* Consider field-level security when updating records

### **Decision Guide: Platform Events vs Queueable**

```
Need async processing?
├── Callout to external system?
│   └── YES → Platform Events (DoWork) with retries
├── Multiple independent subscribers?
│   └── YES → Platform Events (DoWork)
├── External/LWC consumers need to react?
│   └── YES → Platform Events (DoWork)
├── Sequential job chaining?
│   └── YES → Queueable Chain Manager
├── Simple async record update?
│   └── YES → Queueable Chain Manager
└── Single async job, no special requirements?
    └── YES → Queueable Chain Manager (or direct System.enqueueJob)
```

### **Testing Approach**

```apex
@IsTest
private class AsyncErpAccountSyncTest {

    @IsTest
    static void itShouldSyncAccountToErp() {
        // GIVEN an account
        Account account = new Account(Name = 'Test Corp', Industry = 'Technology');
        insert account;

        // WHEN the async worker is published and events are delivered
        Test.startTest();
        new AsyncErpAccountSync(new Set<Id>{ account.Id }).publish(1);
        Test.getEventBus().deliver();
        Test.stopTest();

        // THEN verify the ERP sync occurred
        // (assert via mock callout or custom object log)
    }

    @IsTest
    static void itShouldRetryOnTransientFailure() {
        // GIVEN a failing external service (via mock)
        // WHEN the worker fails and has retries remaining
        // THEN a new event should be published with decremented retry count
    }

    @IsTest
    static void itShouldCallOnExceptionWhenRetriesExhausted() {
        // GIVEN a failing external service with 0 retries
        // WHEN the worker fails
        // THEN onException should be invoked
    }
}
```

### **Trade-offs**

| Benefit | Cost |
| --- | --- |
| Decoupled callout execution | Eventual consistency (not real-time) |
| Built-in retry mechanism | Added complexity in testing |
| Multiple subscribers / fan-out | Requires Platform Event monitoring |
| External system subscription | Cannot return values to caller |
| Immediate publication (with Publish Immediately) | Events cannot be rolled back in Publish Immediately mode |
| Horizontal scalability | 131KB serialization limit per work item |
| Separate governor limit context | Hourly event allocation limits (varies by edition) |
