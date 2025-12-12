# Async-First Pattern with Platform Events (DoWork)

## **Overview**

{% hint style="info" %}
**When to use this pattern:** When trigger operations need to run asynchronously to avoid governor limits, prevent record locking, or ensure immediate trigger completion while deferring complex processing.
{% endhint %}

#### **Purpose**

The DoWork pattern provides a clean abstraction for moving synchronous trigger operations to asynchronous execution using Platform Events. This solves the common challenge of triggers that need to perform operations that would either hit governor limits, cause record locking issues, or need to run after the initial transaction commits.

#### **Context**

In complex Salesforce implementations, triggers often need to:

* Update the same record that triggered them (after auto-number generation)
* Perform callouts to external systems
* Execute operations that exceed governor limits when bulkified
* Avoid holding database locks during long-running operations

Platform Events provide immediate publication (before transaction commit) with separate execution context, making them ideal for async processing.

### **Problem Statement**

#### **The Challenge**

Trigger operations that modify the triggering record or perform complex operations face several challenges:

1. **Record Lock Contention**: Direct updates during `afterInsert` cause `UNABLE_TO_LOCK_ROW` errors in high-concurrency scenarios
2. **Governor Limits**: Complex calculations or callouts may exceed limits when processing bulk records
3. **Auto-Number Timing**: Auto-number fields are not populated until after insert, requiring a separate update
4. **Transaction Rollback Risk**: Long-running operations increase the risk of entire transaction failure

#### **Why Traditional Approaches Fall Short**

* **@future methods**: Cannot accept SObject parameters, limited to 50 calls per transaction
* **Queueable Apex**: Better than @future but still counts against limits and has delay
* **Batch Apex**: Too heavyweight for simple trigger-initiated operations
* **Direct DML in trigger**: Causes record locking and transaction coupling

### **Solution**

#### **Core Concept**

The DoWork pattern uses Platform Events as a lightweight message bus. Work items serialize themselves, publish as events, and a trigger deserializes and executes them in a separate transaction. This provides immediate asynchronous execution with automatic retry capabilities.

#### **Implementation Strategy**

#### **1. Define the Work Interface**

The interface defines the contract that all async work items must fulfill:

```
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

```
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

    /**
     * Override this method with your own logic if required
     * @param e The thrown exception to deal with
     */
    public virtual void onException(Exception e) {
        // Intentionally left blank
    }

    /**
     * Override this method with your own logic if required
     */
    public virtual void onFinally() {
        // Intentionally left blank
    }

    abstract public String getClassName();

    public class WorkException extends Exception {}
}

```

#### **3. Platform Event Trigger Handler**

The trigger deserializes and executes work items with retry logic:

```
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

| Component        | Purpose        | Responsibility                                    |
| ---------------- | -------------- | ------------------------------------------------- |
| `IDoWork`        | Interface      | Defines the contract for async work items         |
| `DoWorkAbstract` | Abstract Base  | Handles serialization and event publication       |
| `DoWork__e`      | Platform Event | Carries serialized work data between transactions |
| `DoWorkTrigger`  | Event Handler  | Deserializes and executes work items with retry   |

### **Implementation Details**

#### **Required Setup**

1. **Create Platform Event**: `DoWork__e` with fields:
   * `Work__c` (Long Text Area, 131072 chars) - Serialized work item
   * `ClassName__c` (Text, 255) - Fully qualified class name
   * `Retries__c` (Number) - Remaining retry attempts
2. **Deploy Abstract Classes**: `IDoWork` interface and `DoWorkAbstract` class
3. **Create Trigger**: `DoWorkTrigger` on `DoWork__e`

#### **Code Structure - Concrete Worker Example**

```
/**
 * Async worker that sets account number from auto-number field.
 * Must run async because auto-number is only available after insert.
 */
public without sharing class AsyncAccountNumberSetter extends DoWorkAbstract {

    private final Set<Id> accountIds;

    public AsyncAccountNumberSetter(Set<Id> accountIds) {
        this.accountIds = accountIds;
    }

    public void doWork() {
        List<Account> accounts = [
            SELECT Auto_Account_Number__c, AccountNumber__c
            FROM Account
            WHERE Id IN :accountIds
            FOR UPDATE  // Obtain a record lock
        ];

        for (Account record : accounts) {
            record.AccountNumber__c = record.Auto_Account_Number__c;
        }

        // Disable triggers to prevent recursion
        fflib_SObjectDomain.getTriggerEvent(AccountTriggerHandler.class).disableAll();
        update accounts;
    }

    public override String getClassName() {
        return 'AsyncAccountNumberSetter';
    }
}

```

#### **Preventing Duplicate Scheduling**

When working with triggers, prevent the same async job from being scheduled multiple times in the same transaction:

```
public with sharing class AccountTriggerHandler extends fflib_SObjectDomain {

    // Static set to track scheduled jobs in current transaction
    private static Set<String> scheduledAsyncTasks = new Set<String>();

    private static void setAccountNumber(IAccounts accounts) {
        // Check if already scheduled
        if (scheduledAsyncTasks.contains('AsyncAccountNumberSetter')) return;

        // Schedule the async work
        new AsyncAccountNumberSetter(accounts.getRecordIds())
            .publish();

        // Mark as scheduled
        scheduledAsyncTasks.add('AsyncAccountNumberSetter');
    }
}

```

#### **Configuration Requirements**

* **Platform Event**: `DoWork__e` must be created with appropriate field limits
* **Permissions**: Users/contexts publishing events need "Publish" permission on `DoWork__e`
* **Dependencies**: Logger utility for error tracking (optional but recommended)

### **Best Practices**

#### **Do's**

* Use `FOR UPDATE` in queries when updating records to prevent concurrent modification issues
* Disable triggers when performing DML to prevent recursion
* Include meaningful class names for debugging and monitoring
* Keep work items as small as possible to stay under the 131KB serialization limit
* Use static tracking sets to prevent duplicate scheduling within transactions
* Implement `onException` for critical workflows that need failure notification

#### **Don'ts**

* Do not serialize large object graphs (query fresh data in `doWork()`)
* Do not rely on execution order - Platform Events may be processed out of order
* Do not store sensitive data in the event payload (it's visible in Event Monitoring)
* Do not use for operations that must complete synchronously with the user action

### **Considerations**

#### **Governor Limits**

* Platform Events have their own limits separate from the triggering transaction
* Event payload is limited to 1MB total, but individual Long Text fields max at 131,072 characters
* Maximum 250,000 platform event allocations per 24 hours (varies by edition)

#### **Performance Impact**

* Platform Events are highly performant for async processing
* Events are published immediately, not at transaction commit
* Parallel processing of events provides horizontal scalability

#### **Security Implications**

* Event payload is stored temporarily and visible in Event Monitoring
* Use appropriate sharing settings (`with sharing` vs `without sharing`) in work classes
* Consider field-level security when updating records

### **Variations**

#### **Variation 1: Base Worker with Common State**

For domain-specific workers, create an intermediate abstract class:

```
public abstract class AsyncAccountWorker extends DoWorkAbstract {
    protected final Set<Id> accountIds;

    public AsyncAccountWorker(Set<Id> accountIds, String className) {
        this.accountIds = accountIds;
        AccountTriggerHandler.scheduledAsyncTasks.add(className);
    }

    abstract public override String getClassName();
}

```

#### **Variation 2: Worker with Retry Configuration**

For operations that may fail transiently:

```
public void scheduleWithRetries() {
    new AsyncExternalSync(recordIds).publish(3);  // Allow 3 retries
}

```

### **Testing Approach**

#### **Unit Test Strategy**

```
@IsTest
private class AsyncAccountNumberSetterTest {

    @IsTest
    static void itShouldSetAccountNumber() {
        // Disable trigger to set up test data cleanly
        fflib_SObjectDomain.getTriggerEvent(AccountTriggerHandler.class).disableAll();
        insert new AutomationBypass__c(TR_Account_Trigger_Disabled__c = true);

        // GIVEN an account without account number
        Account account = new Account(Name = 'Test');
        insert account;

        // WHEN the async worker is published and events are delivered
        System.Test.startTest();
        new AsyncAccountNumberSetter(new Set<Id>{account.Id}).publish();
        System.Test.getEventBus().deliver();  // Critical: deliver events in test
        System.Test.stopTest();

        // THEN the account number should be populated
        Account result = [SELECT AccountNumber__c FROM Account WHERE Id = :account.Id];
        System.Assert.isNotNull(result.AccountNumber__c, 'Account number should be set');
    }
}

```

#### **Test Scenarios**

1. **Happy Path**: Verify work executes successfully and updates records
2. **Retry Logic**: Verify retries occur on transient failures
3. **Error Handling**: Verify `onException` is called when retries are exhausted
4. **Duplicate Prevention**: Verify same work item is not scheduled twice

### **Trade-offs**

| Benefit                         | Cost                                 |
| ------------------------------- | ------------------------------------ |
| Immediate trigger completion    | Eventual consistency (not real-time) |
| Separate governor limit context | Added complexity in testing          |
| Built-in retry mechanism        | Requires Platform Event monitoring   |
| Prevents record locking         | Cannot return values to caller       |
| Horizontal scalability          | Event ordering not guaranteed        |
