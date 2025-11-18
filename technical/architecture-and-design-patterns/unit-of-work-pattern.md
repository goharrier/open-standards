# Unit of Work Pattern in Salesforce

## Intent

The Unit of Work pattern maintains a list of objects affected by a business transaction and coordinates writing out changes and resolving concurrency problems. It ensures all DML operations happen in the correct order and as a single transaction.

## Problem Context

In Salesforce, you face challenges with:
- Complex parent-child relationships requiring specific insert order
- Governor limits on DML statements (150 per transaction)
- Transaction management across multiple service calls
- Bulk operations and trigger recursion
- Maintaining data integrity across related objects

## Core Implementation

### 1. Basic Unit of Work Pattern

```apex
// Unit of Work interface
public interface IUnitOfWork {
    void registerNew(SObject record);
    void registerNew(List<SObject> records);
    void registerNew(SObject record, Schema.SObjectField relatedToField, SObject relatedTo);
    void registerDirty(SObject record);
    void registerDirty(List<SObject> records);
    void registerDeleted(SObject record);
    void registerDeleted(List<SObject> records);
    void commitWork();
}

// Application configuration
public class Application {
    // Define DML order for related objects
    public static final fflib_Application.UnitOfWorkFactory UnitOfWork = 
        new fflib_Application.UnitOfWorkFactory(
            new List<SObjectType>{
                Account.SObjectType,      // Parents first
                Contact.SObjectType,       // Then children
                Case.SObjectType,
                CaseComment.SObjectType,   // Grandchildren last
                Task.SObjectType,
                Event.SObjectType
            }
        );
    
    public static IUnitOfWork newUnitOfWork() {
        return UnitOfWork.newInstance();
    }
}
```

### 2. Understanding Transaction Boundaries

#### The Mental Model

Think of Unit of Work as a **transaction coordinator**. Instead of executing DML immediately, you're building a "change list" that executes atomically at the end.

**Key Questions When Using UoW:**
1. **What's my transaction boundary?** - All related changes that must succeed or fail together
2. **What's the relationship hierarchy?** - Parents must exist before children can reference them
3. **Where are my side effects?** - Emails, events, or custom work that should happen with the transaction

#### Service Layer Pattern

The service layer is where business transactions are orchestrated:

```apex
public void convertQuoteToContract(Id quoteId) {
    IUnitOfWork uow = Application.newUnitOfWork();
    
    // The entire conversion is one transaction
    Quote quote = markQuoteAsAccepted(quoteId, uow);
    Contract contract = createContractFromQuote(quote, uow);
    createContractLines(quote, contract, uow);
    updateRelatedRecords(contract, uow);
    
    // Only NOW does anything actually happen in the database
    uow.commitWork();
}
```

**Why This Matters:**
- **All or nothing** - If contract line creation fails, the quote isn't marked accepted
- **Single DML context** - All operations count as one transaction for limits
- **Predictable state** - Database changes happen at a known point

### 3. Custom Work with IDoWork

#### When to Use Custom Work

The `registerWork` pattern allows you to inject custom logic into the transaction. This is powerful but should be used thoughtfully.

**Use IDoWork When:**
- You need to execute logic **after** records are committed but **within** the transaction
- You're orchestrating complex multi-step operations
- You need to send emails or publish events as part of the transaction
- You want to encapsulate reusable transaction logic

**Don't Use IDoWork When:**
- Simple CRUD operations suffice (use registerNew/Dirty/Deleted)
- Logic should run regardless of transaction success
- You need immediate execution (IDoWork is deferred)

#### Real-World Pattern: Post-Conversion Actions

From production code, here's how custom work enables clean separation:

```apex
// Service layer orchestrates the conversion
public void convertQuoteToContract(Set<Id> quoteIds) {
    IUnitOfWork uow = Application.newUnitOfWork();
    
    // Main conversion logic
    IContracts contracts = createContractsFromQuotes(quoteIds, uow);
    
    // Register post-conversion work
    IQuoteToContractActions afterActions = getAfterConversionActions();
    if (!afterActions.isEmpty()) {
        uow.registerWork(new DoAfterActions(afterActions));
    }
    
    uow.commitWork();
}

// Custom work implementation
private class DoAfterActions implements fflib_SObjectUnitOfWork.IDoWork {
    private final IQuoteToContractActions actions;
    
    public DoAfterActions(IQuoteToContractActions actions) {
        this.actions = actions;
    }
    
    public void doWork() {
        // Executed after records are saved, but within transaction
        actions.execute();
    }
}
```

**Why This Pattern Works:**
- **Separation of concerns** - Core logic vs. post-processing
- **Extensibility** - Easy to add new post-conversion actions
- **Transaction safety** - Actions roll back if they fail
- **Testability** - Mock the work without executing it

### 4. Email Work Pattern

#### The Challenge with Transactional Emails

Sending emails within transactions presents unique challenges:
- Emails shouldn't send if the transaction rolls back
- You want to batch email operations for efficiency
- Email templates need record IDs that don't exist until after insert

#### Solution: Email Work Registration

From production implementations, here's the pattern for transactional emails:

```apex
public void sendInvoiceEmails(IUnitOfWork uow, IInvoices invoices) {
    // Create email work container
    SendEmailWork emailWork = new SendEmailWork();
    
    for (Invoice__c invoice : invoices.getRecords()) {
        // Build email with template
        Messaging.SingleEmailMessage email = new Messaging.SingleEmailMessage();
        email.setTargetObjectId(invoice.ContactId__c);
        email.setTemplateId(getInvoiceTemplateId());
        email.setWhatId(invoice.Id);  // ID will exist when email sends
        email.setSaveAsActivity(false);
        
        emailWork.registerEmail(email);
    }
    
    // Register work only if there are emails
    if (emailWork.hasEmails()) {
        uow.registerWork(emailWork);
    }
}

// Reusable email work implementation
private class SendEmailWork implements fflib_SObjectUnitOfWork.IDoWork {
    private List<Messaging.Email> emails = new List<Messaging.Email>();
    
    public void registerEmail(Messaging.Email email) {
        emails.add(email);
    }
    
    public Boolean hasEmails() {
        return !emails.isEmpty();
    }
    
    public void doWork() {
        if (!emails.isEmpty()) {
            Messaging.sendEmail(emails);
        }
    }
}
```

**Key Benefits:**
- **Transaction safety** - Emails only send if DML succeeds
- **Bulk efficiency** - All emails sent in one call
- **Template support** - WhatId references exist when emails send
- **Reusability** - Same pattern works for any email scenario

#### Understanding Execution Order

The Unit of Work executes operations in this sequence:
1. DML operations (in configured SObject order)
2. Registered work (IDoWork implementations)
3. Email dispatch (if configured)
4. Event publishing (if configured)

This matters because:
- Your custom work can reference saved record IDs
- Emails can use merge fields from committed records
- Events publish after data is persisted
- Everything rolls back together on failure


### 5. Testing with Mocked Unit of Work

#### Why Mock the Unit of Work?

Mocking UoW in tests provides several benefits:
- **Faster tests** - No actual DML operations
- **Focused testing** - Test logic, not database operations
- **Predictable behavior** - No side effects or triggers
- **Verify interactions** - Ensure correct methods are called

#### The Mocking Pattern

From production test suites, here's the standard approach:

```apex
@IsTest
private class QuestionnaireServiceTest {
    
    @IsTest
    static void testSaveAnswers() {
        // Setup mocks
        fflib_ApexMocks mocks = new fflib_ApexMocks();
        fflib_ISObjectUnitOfWork uowMock = 
            (fflib_ISObjectUnitOfWork) mocks.mock(fflib_ISObjectUnitOfWork.class);
        
        // Inject mock into application
        Application.UnitOfWork.setMock(uowMock);
        
        // Prepare test data
        DamageAnswer__c answer = new DamageAnswer__c(
            Question__c = 'Test Question',
            Answer__c = 'Test Answer'
        );
        
        // Execute service method
        Test.startTest();
        QuestionnaireService.saveAnswers(new List<DamageAnswer__c>{answer});
        Test.stopTest();
        
        // Verify interactions
        ((fflib_ISObjectUnitOfWork) mocks.verify(uowMock, 1))
            .registerUpsert(new List<DamageAnswer__c>{answer});
        ((fflib_ISObjectUnitOfWork) mocks.verify(uowMock, 1))
            .commitWork();
    }
    
    @IsTest
    static void testErrorHandling() {
        fflib_ApexMocks mocks = new fflib_ApexMocks();
        fflib_ISObjectUnitOfWork uowMock = 
            (fflib_ISObjectUnitOfWork) mocks.mock(fflib_ISObjectUnitOfWork.class);
        
        // Configure mock to throw exception
        ((fflib_ISObjectUnitOfWork) mocks.doThrowWhen(
            new DmlException('Test Error'), 
            uowMock
        )).commitWork();
        
        Application.UnitOfWork.setMock(uowMock);
        
        // Verify service handles error appropriately
        try {
            QuestionnaireService.process();
            System.assert(false, 'Should have thrown exception');
        } catch (ServiceException e) {
            System.assert(e.getMessage().contains('Test Error'));
        }
    }
}
```

#### Testing Custom Work

When testing IDoWork implementations:

```apex
@IsTest
static void testCustomWork() {
    // Test the work in isolation
    SendEmailWork emailWork = new SendEmailWork();
    emailWork.registerEmail(buildTestEmail());
    
    Test.startTest();
    emailWork.doWork();  // Direct testing
    Test.stopTest();
    
    // Verify email was sent (check limits or mock email)
    System.assertEquals(1, Limits.getEmailInvocations());
}

@IsTest
static void testWorkRegistration() {
    fflib_ApexMocks mocks = new fflib_ApexMocks();
    fflib_ISObjectUnitOfWork uowMock = 
        (fflib_ISObjectUnitOfWork) mocks.mock(fflib_ISObjectUnitOfWork.class);
    
    // Verify work is registered
    EmailService.sendInvoiceEmails(uowMock, testInvoices);
    
    // Verify registerWork was called
    ((fflib_ISObjectUnitOfWork) mocks.verify(uowMock, 1))
        .registerWork((fflib_SObjectUnitOfWork.IDoWork) fflib_Match.anyObject());
}
```

**Testing Strategy:**
1. **Mock for logic testing** - Verify service behavior
2. **Real UoW for integration** - Test actual database operations
3. **Isolate custom work** - Test IDoWork implementations directly
4. **Verify error handling** - Ensure graceful failure

## Implementation Patterns

### 1. Nested Unit of Work

```apex
public class ComplexService {
    
    public void processComplexTransaction() {
        IUnitOfWork outerUow = Application.newUnitOfWork();
        
        // Process main entities
        processMainEntities(outerUow);
        
        // Process sub-transactions
        for (SubProcess sp : getSubProcesses()) {
            IUnitOfWork innerUow = Application.newUnitOfWork();
            processSubTransaction(sp, innerUow);
            innerUow.commitWork(); // Commit sub-transaction
        }
        
        // Commit main transaction
        outerUow.commitWork();
    }
}
```

### 2. Conditional Registration

```apex
public class ConditionalService {
    
    public void processWithConditions(List<Lead> leads) {
        IUnitOfWork uow = Application.newUnitOfWork();
        
        for (Lead l : leads) {
            if (l.Status == 'Qualified') {
                // Convert to account
                Account acc = convertToAccount(l);
                uow.registerNew(acc);
                
                // Mark lead as converted
                l.IsConverted = true;
                uow.registerDirty(l);
            } else if (l.Status == 'Disqualified') {
                // Delete lead
                uow.registerDeleted(l);
            } else {
                // Just update
                uow.registerDirty(l);
            }
        }
        
        uow.commitWork();
    }
}
```

## Benefits

- **Transaction Management**: All-or-nothing commits
- **Bulk Operations**: Automatic bulkification of DML
- **Relationship Management**: Handles parent-child relationships
- **Governor Limit Optimization**: Minimizes DML statements
- **Testability**: Easy to mock for unit tests
- **Separation of Concerns**: Business logic separate from persistence

## Trade-offs

- **Memory Usage**: Holds records in memory until commit
- **Complexity**: Additional abstraction layer
- **Debugging**: Can be harder to trace DML operations
- **Learning Curve**: Team needs to understand pattern

## Best Practices

### 1. Define Clear Object Order
```apex
// Parents → Children → Grandchildren
new List<SObjectType>{
    Account.SObjectType,        // Level 0
    Contact.SObjectType,        // Level 1
    Case.SObjectType,           // Level 1
    CaseComment.SObjectType     // Level 2
}
```

### 2. Single Unit of Work Per Transaction
```apex
// ✅ GOOD: One UoW for transaction
public void processOrder() {
    IUnitOfWork uow = Application.newUnitOfWork();
    // All operations
    uow.commitWork();
}

// ❌ BAD: Multiple UoWs can cause partial commits
public void processOrder() {
    IUnitOfWork uow1 = Application.newUnitOfWork();
    // Some operations
    uow1.commitWork();
    
    IUnitOfWork uow2 = Application.newUnitOfWork();
    // More operations - if this fails, uow1 changes persist
    uow2.commitWork();
}
```

### 3. Pass Unit of Work to Methods
```apex
public void processOrder(OrderRequest request) {
    IUnitOfWork uow = Application.newUnitOfWork();
    
    createCustomer(request.customer, uow);
    createOrder(request.order, uow);
    updateInventory(request.items, uow);
    
    uow.commitWork();
}

private void createCustomer(CustomerData data, IUnitOfWork uow) {
    // Use passed UoW, don't create new one
    Account customer = new Account(Name = data.name);
    uow.registerNew(customer);
}
```

## Anti-Patterns to Avoid

### 1. Premature Commits
```apex
// ❌ BAD: Committing inside loop
for (Account acc : accounts) {
    IUnitOfWork uow = Application.newUnitOfWork();
    uow.registerDirty(acc);
    uow.commitWork(); // DML in loop!
}

// ✅ GOOD: Single commit
IUnitOfWork uow = Application.newUnitOfWork();
for (Account acc : accounts) {
    uow.registerDirty(acc);
}
uow.commitWork(); // One DML
```

### 2. Mixing Direct DML
```apex
// ❌ BAD: Mixing patterns
IUnitOfWork uow = Application.newUnitOfWork();
uow.registerNew(account);
insert contact; // Direct DML - not in transaction!
uow.commitWork();

// ✅ GOOD: Consistent pattern
IUnitOfWork uow = Application.newUnitOfWork();
uow.registerNew(account);
uow.registerNew(contact);
uow.commitWork();
```

## Real-World Usage

Production systems use Unit of Work for:
- **Complex order processing** with multiple related objects
- **Data migration** maintaining relationships
- **Trigger operations** managing related records
- **Integration responses** updating multiple objects
- **Bulk operations** from batch processes

The pattern is essential for maintaining data integrity while respecting Salesforce governor limits.