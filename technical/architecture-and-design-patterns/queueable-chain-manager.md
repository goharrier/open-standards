# Queueable Chain Manager Pattern

## **Overview**

{% hint style="info" %}
**When to use this pattern:** When you need to run multiple async jobs sequentially from within an already-async context (queueable, batch, future), or when trigger logic needs simple async execution without the overhead of Platform Events.
{% endhint %}

#### **Purpose**

The Queueable Chain Manager solves the "Too many queueable jobs" governor limit by collecting jobs and chaining them sequentially using the `System.Finalizer` pattern. Each job runs in its own transaction with full governor limits, and the chain continues even if an individual job fails.

#### **Context**

In Salesforce, async contexts (Queueable, Batch, Future) are limited to enqueueing **1 queueable job** per transaction. When multiple pieces of work need to run asynchronously — e.g., a trigger handler that needs to update records, recalculate rollups, and sync related data — you quickly hit this limit.

#### **When to Use This vs Platform Events**

| Scenario | Use Queueable Chain Manager | Use [Platform Events (DoWork)](async-first-pattern-with-platform-events-dowork.md) |
| --- | --- | --- |
| Async record updates | Yes | No — overkill |
| Sequential job chaining | Yes | No — PE doesn't guarantee order |
| Multiple jobs from async context | Yes | Possible, but chain manager is simpler |
| External callouts with retry | No | Yes — better decoupling |
| Multiple independent subscribers | No | Yes — PE's core strength |
| External/LWC consumers | No | Yes — CometD/Empapi |

### **Problem Statement**

#### **The Challenge**

1. **Queueable Limit in Async Context**: You can only enqueue 1 job from a Queueable, Batch, or Future context
2. **Job Ordering**: Multiple async operations may need to run in sequence
3. **Error Isolation**: One failing job should not prevent subsequent jobs from running
4. **Dynamic Job Addition**: Jobs registered during execution of another job need to be picked up

#### **Why Traditional Approaches Fall Short**

* **@future methods**: Cannot accept SObject parameters, limited to 50 calls per transaction, no chaining
* **Direct System.enqueueJob()**: Throws `LimitException` when called more than once in async context
* **Batch Apex**: Too heavyweight for simple operations, no chaining support
* **Platform Events**: No ordering guarantee, adds serialization overhead, consumes hourly event allocations

### **Solution**

#### **Core Concept**

The `QueueableChainManager` collects `Queueable` jobs into a pending list. When the parent async job completes, it calls `enqueueChain()` which wraps each job in a `ChainableQueueableWrapper`. The wrapper uses `System.Finalizer` to enqueue the next job after the current one commits, creating a sequential chain where each job gets its own transaction and full governor limits.

#### **Architecture**

```
Trigger / Service
    │
    ├── QueueableChainManager.registerJob(job1)  ──→ [pendingJobs list]
    ├── QueueableChainManager.registerJob(job2)  ──→ [pendingJobs list]
    └── QueueableChainManager.registerJob(job3)  ──→ [pendingJobs list]
                                                          │
                                                  enqueueChain()
                                                          │
                                                          ▼
                                            ChainableQueueableWrapper(job1)
                                                    │         │
                                              execute(job1)   attachFinalizer([job2, job3])
                                                                      │
                                                                      ▼
                                                            ChainFinalizer.execute()
                                                                      │
                                                                      ▼
                                                      ChainableQueueableWrapper(job2)
                                                              │         │
                                                        execute(job2)   attachFinalizer([job3])
                                                                                │
                                                                                ▼
                                                                      ... and so on
```

#### **Implementation**

```apex
public class QueueableChainManager {

    private static List<Queueable> pendingJobs = new List<Queueable>();

    /**
     * Registers a queueable job.
     * - In async context (queueable/batch/future): adds to pending chain
     * - In synchronous context: enqueues directly via System.enqueueJob()
     * @return true if added to chain, false if enqueued directly
     */
    public static Boolean registerJob(Queueable job) {
        if (System.isQueueable() || System.isBatch() || System.isFuture()) {
            pendingJobs.add(job);
            return true;
        } else {
            System.enqueueJob(job);
            return false;
        }
    }

    /**
     * Starts the queueable chain.
     * Call this at the end of the parent queueable's execute() method.
     */
    public static void enqueueChain() {
        if (pendingJobs.isEmpty()) {
            return;
        }

        List<Queueable> jobsToChain = new List<Queueable>(pendingJobs);
        pendingJobs.clear();

        enqueueWithChain(jobsToChain);
    }

    @TestVisible
    private static void enqueueWithChain(List<Queueable> jobs) {
        if (jobs.isEmpty()) {
            return;
        }

        Queueable currentJob = jobs.remove(0);
        ChainableQueueableWrapper wrapper = new ChainableQueueableWrapper(currentJob, jobs);
        System.enqueueJob(wrapper);
    }

    public static Boolean hasPendingJobs() {
        return !pendingJobs.isEmpty();
    }

    public static Integer getPendingJobCount() {
        return pendingJobs.size();
    }

    /**
     * Clears pending jobs.
     * Call when a transaction fails and will be retried — static variables
     * don't roll back with the transaction.
     */
    public static void clearPendingJobs() {
        pendingJobs.clear();
    }

    private static String getTypeName(Object obj) {
        if (obj == null) return 'null';
        return String.valueOf(obj).substringBefore(':');
    }

    /**
     * Wraps a Queueable and uses System.Finalizer to chain to the next job.
     * Finalizer is attached in the finally block AFTER execution because:
     * 1. Finalizers are serialized when attached — later list modifications won't be seen
     * 2. Finally block ensures chain continues even if the job throws an exception
     * 3. Any jobs dynamically added during execution are included
     */
    @TestVisible
    private class ChainableQueueableWrapper implements Queueable {

        private Queueable currentJob;
        private List<Queueable> remainingJobs;

        public ChainableQueueableWrapper(Queueable currentJob, List<Queueable> remainingJobs) {
            this.currentJob = currentJob;
            this.remainingJobs = remainingJobs;
        }

        public void execute(QueueableContext context) {
            pendingJobs.clear();

            try {
                currentJob.execute(context);
            } finally {
                attachFinalizerForRemainingJobs();
            }
        }

        private void attachFinalizerForRemainingJobs() {
            List<Queueable> allRemainingJobs = new List<Queueable>(remainingJobs);
            if (!pendingJobs.isEmpty()) {
                allRemainingJobs.addAll(pendingJobs);
                pendingJobs.clear();
            }

            if (allRemainingJobs.isEmpty()) {
                return;
            }

            try {
                System.attachFinalizer(new ChainFinalizer(allRemainingJobs));
            } catch (Exception e) {
                System.debug(LoggingLevel.ERROR,
                    'ChainableQueueableWrapper: Failed to attach finalizer. ' +
                    'Chain will not continue. Error: ' + e.getMessage());
            }
        }
    }

    /**
     * Finalizer that enqueues the next job after the current job's transaction
     * commits (success) or rolls back (failure).
     * Chain continues regardless of previous job outcome.
     */
    @TestVisible
    private class ChainFinalizer implements Finalizer {

        private List<Queueable> remainingJobs;

        public ChainFinalizer(List<Queueable> remainingJobs) {
            this.remainingJobs = remainingJobs;
        }

        public void execute(FinalizerContext context) {
            if (remainingJobs.isEmpty()) {
                return;
            }

            try {
                enqueueWithChain(remainingJobs);
            } catch (Exception e) {
                System.debug(LoggingLevel.ERROR,
                    'ChainFinalizer: Failed to enqueue next job. ' +
                    'Chain will not continue. Error: ' + e.getMessage());
            }
        }
    }
}
```

### **Usage Examples**

#### **From a Trigger Handler**

In synchronous context, `registerJob` enqueues directly — no need to call `enqueueChain()`:

```apex
public class AccountTriggerHandler extends fflib_SObjectDomain {

    public override void onAfterInsert() {
        QueueableChainManager.registerJob(new RecalculateRollupsJob(getRecordIds()));
        QueueableChainManager.registerJob(new SyncRelatedContactsJob(getRecordIds()));
        // Both jobs enqueue directly (synchronous context allows multiple enqueues)
    }
}
```

#### **From Within a Queueable (Chaining)**

In async context, jobs are collected and chained:

```apex
public class RecalculateRollupsJob implements Queueable {

    private Set<Id> accountIds;

    public RecalculateRollupsJob(Set<Id> accountIds) {
        this.accountIds = accountIds;
    }

    public void execute(QueueableContext context) {
        // Do the rollup work
        List<Account> accounts = [SELECT Id FROM Account WHERE Id IN :accountIds];
        // ... perform calculations ...
        update accounts;

        // Register follow-up jobs (these go to pendingJobs since we're in async context)
        QueueableChainManager.registerJob(new NotifyStakeholdersJob(accountIds));

        // IMPORTANT: call enqueueChain() at the end to start the chain
        QueueableChainManager.enqueueChain();
    }
}
```

#### **From Batch Apex**

```apex
public class AccountBatchProcess implements Database.Batchable<SObject> {

    public void execute(Database.BatchableContext bc, List<Account> scope) {
        // Process batch scope...

        // Register async follow-up work
        Set<Id> processedIds = new Map<Id, Account>(scope).keySet();
        QueueableChainManager.registerJob(new PostProcessingJob(processedIds));

        // Chain at end of execute
        QueueableChainManager.enqueueChain();
    }
}
```

### **The System.Finalizer Pattern**

The `System.Finalizer` interface (beta in Spring '20, GA in Summer '21 / API v52) is the foundation of this pattern. Understanding how it works is essential.

#### **What Is a Finalizer?**

A Finalizer is an Apex class implementing the `Finalizer` interface. You attach it to a Queueable job via `System.attachFinalizer()`. After the Queueable transaction completes — whether it commits successfully or rolls back due to an unhandled exception — the platform executes the Finalizer in a **separate transaction** with its own governor limits.

```apex
public class MyFinalizer implements Finalizer {
    public void execute(FinalizerContext context) {
        String parentJobId = context.getAsyncApexJobId();
        ParentJobResult result = context.getResult();

        if (result == ParentJobResult.SUCCESS) {
            // Parent committed successfully
        } else if (result == ParentJobResult.UNHANDLED_EXCEPTION) {
            // Parent rolled back — exception available via context.getException()
            Exception e = context.getException();
        }
    }
}
```

#### **Key Finalizer Rules**

| Rule | Detail |
| --- | --- |
| **One per Queueable** | Only 1 Finalizer can be attached per Queueable execution. Calling `attachFinalizer()` again overwrites the previous one. |
| **Serialized on attach** | The Finalizer instance is serialized at the moment `attachFinalizer()` is called. Any later mutations to the Finalizer's fields are **not** reflected. This is why we attach in the `finally` block — after all dynamic jobs have been collected. |
| **Separate transaction** | The Finalizer runs in its own transaction with fresh governor limits — it can enqueue new Queueable jobs, perform DML, make callouts, etc. |
| **Runs on success AND failure** | The Finalizer fires regardless of whether the parent Queueable succeeded or threw an exception. This is what makes error-resilient chaining possible. |
| **Cannot be attached outside Queueable** | `System.attachFinalizer()` only works inside a Queueable's `execute()` method. Calling it elsewhere throws an exception. |

#### **Why Finalizer Instead of Recursive Enqueue**

Before Finalizers, the common pattern was to call `System.enqueueJob()` inside a Queueable's `execute()` to chain. This had problems:

* If the parent job fails, the recursive enqueue is rolled back — chain breaks
* You must enqueue before the job completes, which means the child job might start while the parent is still executing
* Governor limit sharing between parent setup and child enqueue

The Finalizer pattern solves all of these because it runs **after** the parent transaction is fully resolved.

#### **How This Pattern Uses Finalizers**

The `QueueableChainManager` uses the Finalizer as the chaining mechanism:

1. `ChainableQueueableWrapper` executes the current job
2. In the `finally` block, it collects remaining jobs + any dynamically registered jobs
3. It attaches a `ChainFinalizer` containing the remaining job list
4. The Finalizer serializes the job list at attach time
5. After the transaction resolves, the Finalizer's `execute()` calls `enqueueWithChain()` for the next job
6. This creates a sequential chain where each job gets a clean transaction

#### **Configurable Chain Depth (Spring '24)**

As of Spring '24 (API v60), admins can configure the maximum depth of chained Queueable jobs via **Setup → Apex Settings → Maximum Depth of Chained Queueable Jobs**. Previously, developer/trial orgs had a hard limit of 5 chained jobs. This setting removes that constraint for production use cases with long chains.

#### **Delay Parameter**

`System.enqueueJob(queueable, delay)` accepts an optional delay in minutes before the job begins execution. This can be useful for rate-limiting chains or implementing backoff strategies.

### **Key Design Decisions**

#### **Why Static List for Pending Jobs**

Static variables persist for the duration of a transaction. Jobs registered anywhere in the call stack (trigger handlers, services, domain classes) are collected in one place. The chain manager dequeues and chains them at the end.

**Important caveat**: Static variables do **not** roll back with DML rollback. If you catch an exception and retry, call `clearPendingJobs()` first to avoid duplicate job execution.

#### **Dynamic Job Addition**

Jobs registered during execution of a chained job (via `registerJob()`) are automatically appended to the chain. The wrapper checks `pendingJobs` in its `finally` block and includes any new jobs when attaching the finalizer.

### **Best Practices**

#### **Do's**

* Always call `enqueueChain()` at the end of your queueable's `execute()` method
* Call `clearPendingJobs()` when retrying after a caught exception
* Keep individual jobs focused — one responsibility per job
* Use `registerJob()` consistently instead of mixing with `System.enqueueJob()`
* Log job execution for observability (job type, duration, success/failure)

#### **Don'ts**

* Do not call `enqueueChain()` from synchronous context — it's only needed in async context
* Do not assume job execution order matches registration order across transactions (within a single chain, order is preserved)
* Do not store large state in Queueable instance variables — they must be serializable for the Finalizer
* Do not use for callout-heavy workflows where Platform Events provide better retry semantics

### **Considerations**

#### **Governor Limits**

* Each chained job runs in its own transaction with full governor limits
* The `System.Finalizer` counts as 1 finalizer per queueable (limit: 1 per queueable)
* Long chains will eventually hit the async job queue capacity of the org

#### **Error Handling**

* A failing job does **not** break the chain — the Finalizer fires regardless
* Failed jobs' DML is rolled back, but the chain continues
* If the Finalizer itself fails to enqueue (e.g., serialization error), the chain stops — monitor for this
* Consider adding a logging Queueable at the end of critical chains to confirm completion

#### **Serialization**

The `System.attachFinalizer()` call serializes the Finalizer instance (including the remaining jobs list it holds). This is platform behavior, not something our code does explicitly. This has important implications:

* All `Queueable` classes in the chain must be JSON-serializable at the time `attachFinalizer()` is called
* Avoid non-serializable fields (HttpRequest, Blob, transient references) — pass IDs and query fresh data in `execute()`
* If a class cannot be serialized, the `attachFinalizer` call will fail and the chain stops
* The Finalizer is attached in the `finally` block specifically so that any dynamically added jobs are included before serialization occurs

### **Testing Approach**

```apex
@IsTest
private class QueueableChainManagerTest {

    @IsTest
    static void itShouldEnqueueDirectlyInSyncContext() {
        // GIVEN a queueable job
        TestQueueable job = new TestQueueable();

        // WHEN registered in synchronous context
        Boolean addedToChain = QueueableChainManager.registerJob(job);

        // THEN it should be enqueued directly
        System.Assert.isFalse(addedToChain, 'Should enqueue directly in sync context');
        System.Assert.areEqual(0, QueueableChainManager.getPendingJobCount());
    }

    @IsTest
    static void itShouldChainJobsInAsyncContext() {
        // GIVEN multiple jobs registered from a queueable context
        // WHEN enqueueChain() is called
        Test.startTest();
        System.enqueueJob(new ParentQueueable());
        Test.stopTest();

        // THEN all jobs should have executed (verify via side effects)
    }

    @IsTest
    static void itShouldContinueChainAfterJobFailure() {
        // GIVEN a chain where the first job throws an exception
        // WHEN the chain executes
        // THEN subsequent jobs should still run
    }

    private class TestQueueable implements Queueable {
        public void execute(QueueableContext context) {
            // Test implementation
        }
    }

    private class ParentQueueable implements Queueable {
        public void execute(QueueableContext context) {
            QueueableChainManager.registerJob(new TestQueueable());
            QueueableChainManager.registerJob(new TestQueueable());
            QueueableChainManager.enqueueChain();
        }
    }
}
```

### **Trade-offs**

| Benefit | Cost |
| --- | --- |
| Avoids "Too many queueable jobs" error | Sequential execution (no parallelism) |
| Each job gets full governor limits | Chain stops if Finalizer serialization fails |
| Chain continues through job failures | Static variable state doesn't roll back |
| Dynamic job addition during execution | Debugging requires tracing across transactions |
| No Platform Event overhead or daily limits | No external subscriber support |
| Simple API — just `registerJob()` and `enqueueChain()` | Must remember to call `enqueueChain()` in async context |
