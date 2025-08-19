# Trigger Modularization with Trigger Actions Framework

## Overview

This guide demonstrates how to migrate from monolithic trigger handlers to a modular, metadata-driven trigger actions framework that enables small, focused, testable trigger logic units.

## The Problem with Traditional Trigger Handlers

Most Salesforce implementations suffer from:
- **Monolithic trigger handlers** with hundreds/thousands of lines
- **Mixed responsibilities** in a single class
- **Poor testability** - need to test entire handler
- **Merge conflicts** when multiple developers work on same handler
- **No configuration** - everything is hardcoded
- **Difficult debugging** - hard to isolate issues

### Typical Monolithic Pattern

```apex
// What we typically see - massive trigger handler classes
public class AccountTriggerHandler extends TriggerHandler {
    
    public override void beforeInsert() {
        // 50+ lines of validation logic
        validateAccounts();
        
        // 30+ lines of defaulting logic
        setDefaultValues();
        
        // 40+ lines of formatting logic
        formatPhoneNumbers();
        
        // 60+ lines of duplicate checking
        checkForDuplicates();
    }
    
    public override void afterInsert() {
        // 100+ lines creating related records
        createDefaultContacts();
        createTeamMembers();
        createShares();
        
        // 80+ lines of integration logic
        sendToExternalSystem();
        publishPlatformEvents();
        
        // 40+ lines of notification logic
        sendEmailAlerts();
    }
    
    public override void beforeUpdate() {
        // 200+ lines of various business logic
        validateStatusTransitions();
        calculateScores();
        updateDerivedFields();
        enforceBusinessRules();
        // ... and on and on
    }
    
    // Often 1000+ lines total in production handlers
}
```

## The Trigger Actions Solution

### Core Concept

Instead of one large handler, create **small, focused trigger action classes** that each do ONE thing, configured through **Custom Metadata**.

### Architecture Overview

```
Trigger → fflib_TriggerHandler → Custom Metadata → Individual Action Classes
```

## Implementation Pattern

### Step 1: Simple Trigger

```apex
// The trigger is now just a dispatcher
trigger CasesTrigger on Case (
    before insert, before update, before delete,
    after insert, after update, after delete, after undelete
) {
    new fflib_TriggerHandler().run();
}
```

### Step 2: Individual Trigger Actions

Each action is a small, focused class:

```apex
// Action 1: Send escalation emails
public class TA_Case_SendEscalationEmails extends fflib_TriggerAction {
    
    public override void onAfterUpdate() {
        List<Case> escalatedCases = getEscalatedCases();
        
        if (escalatedCases.isEmpty()) return;
        
        publishEscalationEvents(escalatedCases);
    }
    
    private List<Case> getEscalatedCases() {
        return (List<Case>) triggerContext.getChangedRecords(
            new Set<SObjectField>{Case.SubStatus__c}
        ).stream()
            .filter(c -> c.SubStatus__c == 'Escalated')
            .collect(Collectors.toList());
    }
    
    private void publishEscalationEvents(List<Case> cases) {
        List<Case_Escalation_Event__e> events = new List<Case_Escalation_Event__e>();
        
        for (Case c : cases) {
            events.add(new Case_Escalation_Event__e(
                CaseId__c = c.Id
            ));
        }
        
        EventBus.publish(events);
    }
}

// Action 2: Format phone numbers
public class TA_Case_FormatPhoneNumbers extends fflib_TriggerAction {
    
    public override void onBeforeInsert() {
        formatPhones();
    }
    
    public override void onBeforeUpdate() {
        formatPhones();
    }
    
    private void formatPhones() {
        for (Case c : (List<Case>) triggerContext.newList) {
            if (String.isNotBlank(c.ContactPhone)) {
                c.ContactPhone = PhoneFormatter.format(c.ContactPhone);
            }
        }
    }
}

// Action 3: Set default values
public class TA_Case_SetDefaults extends fflib_TriggerAction {
    
    public override void onBeforeInsert() {
        for (Case c : (List<Case>) triggerContext.newList) {
            if (c.Priority == null) {
                c.Priority = 'Medium';
            }
            if (c.Origin == null) {
                c.Origin = 'Web';
            }
        }
    }
}

// Action 4: Track field history
public class TA_Case_TrackHistory extends fflib_TriggerAction {
    
    public override void onAfterUpdate() {
        List<Field_History__c> histories = new List<Field_History__c>();
        
        for (Case newCase : (List<Case>) triggerContext.newList) {
            Case oldCase = (Case) triggerContext.oldMap.get(newCase.Id);
            
            if (newCase.Status != oldCase.Status) {
                histories.add(createHistory(
                    newCase.Id, 
                    'Status', 
                    oldCase.Status, 
                    newCase.Status
                ));
            }
        }
        
        if (!histories.isEmpty()) {
            insert histories;
        }
    }
    
    private Field_History__c createHistory(Id recordId, String field, Object oldVal, Object newVal) {
        return new Field_History__c(
            Record_Id__c = recordId,
            Field_Name__c = field,
            Old_Value__c = String.valueOf(oldVal),
            New_Value__c = String.valueOf(newVal),
            Changed_Date__c = System.now()
        );
    }
}
```

### Step 3: Configure Actions in Custom Metadata

```xml
<!-- fflib_TriggerAction.Case_SendEscalationEmails.md-meta.xml -->
<CustomMetadata xmlns="http://soap.sforce.com/2006/04/metadata">
    <label>Case - Send Escalation Emails</label>
    <protected>false</protected>
    <values>
        <field>ObjectType__c</field>
        <value>Case</value>
    </values>
    <values>
        <field>ImplementationType__c</field>
        <value>TA_Case_SendEscalationEmails</value>
    </values>
    <values>
        <field>AfterUpdate__c</field>
        <value>true</value>
    </values>
    <values>
        <field>Sequence__c</field>
        <value>10</value>
    </values>
    <values>
        <field>Active__c</field>
        <value>true</value>
    </values>
</CustomMetadata>

<!-- fflib_TriggerAction.Case_FormatPhoneNumbers.md-meta.xml -->
<CustomMetadata xmlns="http://soap.sforce.com/2006/04/metadata">
    <label>Case - Format Phone Numbers</label>
    <protected>false</protected>
    <values>
        <field>ObjectType__c</field>
        <value>Case</value>
    </values>
    <values>
        <field>ImplementationType__c</field>
        <value>TA_Case_FormatPhoneNumbers</value>
    </values>
    <values>
        <field>BeforeInsert__c</field>
        <value>true</value>
    </values>
    <values>
        <field>BeforeUpdate__c</field>
        <value>true</value>
    </values>
    <values>
        <field>Sequence__c</field>
        <value>5</value>
    </values>
    <values>
        <field>Active__c</field>
        <value>true</value>
    </values>
</CustomMetadata>
```

## Advanced Patterns

### 1. Conditional Trigger Actions

```apex
public class TA_Case_ConditionalAction extends fflib_TriggerAction {
    
    // Only run for specific record types
    public override Boolean shouldRun() {
        if (triggerContext.isInsert) {
            return hasTargetRecordType();
        }
        return false;
    }
    
    private Boolean hasTargetRecordType() {
        Set<Id> targetRecordTypes = new Set<Id>{
            Schema.SObjectType.Case.getRecordTypeInfosByDeveloperName()
                .get('Support_Case').getRecordTypeId()
        };
        
        for (Case c : (List<Case>) triggerContext.newList) {
            if (targetRecordTypes.contains(c.RecordTypeId)) {
                return true;
            }
        }
        return false;
    }
    
    public override void onAfterInsert() {
        // Action logic here
    }
}
```

### 2. Stateful Trigger Actions

```apex
// Track state across trigger contexts
public class TA_Case_PreventRecursion extends fflib_TriggerAction {
    
    private static Set<Id> processedIds = new Set<Id>();
    
    public override void onAfterUpdate() {
        List<Case> toProcess = new List<Case>();
        
        for (Case c : (List<Case>) triggerContext.newList) {
            if (!processedIds.contains(c.Id)) {
                toProcess.add(c);
                processedIds.add(c.Id);
            }
        }
        
        if (!toProcess.isEmpty()) {
            processRecords(toProcess);
        }
    }
}
```

### 3. Async Trigger Actions

```apex
public class TA_Case_AsyncProcessing extends fflib_TriggerAction {
    
    public override void onAfterInsert() {
        // Collect IDs for async processing
        Set<Id> caseIds = triggerContext.newMap.keySet();
        
        // Enqueue async job
        System.enqueueJob(new AsyncProcessor(caseIds));
    }
    
    public class AsyncProcessor implements Queueable {
        private Set<Id> recordIds;
        
        public AsyncProcessor(Set<Id> recordIds) {
            this.recordIds = recordIds;
        }
        
        public void execute(QueueableContext context) {
            // Async processing logic
        }
    }
}
```

### 4. Trigger Action with Dependencies

```apex
public class TA_Case_WithDependencies extends fflib_TriggerAction {
    
    @TestVisible
    private ICaseService caseService {
        get {
            if (caseService == null) {
                caseService = (ICaseService) Application.Service.newInstance(
                    ICaseService.class
                );
            }
            return caseService;
        }
        set;
    }
    
    public override void onAfterUpdate() {
        // Use injected service
        caseService.processUpdatedCases(
            triggerContext.newMap.keySet()
        );
    }
}
```

## Testing Trigger Actions

### Individual Action Testing

```apex
@IsTest
private class TA_Case_SendEscalationEmailsTest {
    
    @IsTest
    static void testEscalationEmailsSent() {
        // Create test data
        Case testCase = new Case(
            Status = 'New',
            SubStatus__c = 'In Progress'
        );
        insert testCase;
        
        // Update to trigger action
        testCase.SubStatus__c = 'Escalated';
        
        Test.startTest();
        update testCase;
        Test.stopTest();
        
        // Verify platform event was published
        List<Case_Escalation_Event__e> events = [
            SELECT CaseId__c 
            FROM Case_Escalation_Event__e 
            WHERE CaseId__c = :testCase.Id
        ];
        
        System.assertEquals(1, events.size());
    }
    
    @IsTest
    static void testNoEscalationForOtherStatus() {
        Case testCase = new Case(
            Status = 'New',
            SubStatus__c = 'In Progress'
        );
        insert testCase;
        
        testCase.SubStatus__c = 'Resolved';
        
        Test.startTest();
        update testCase;
        Test.stopTest();
        
        // Verify no event published
        // Test passes if no exception
    }
}
```

### Testing with Mocks

```apex
@IsTest
private class TA_Case_WithDependenciesTest {
    
    @IsTest
    static void testServiceCalled() {
        fflib_ApexMocks mocks = new fflib_ApexMocks();
        ICaseService mockService = (ICaseService) mocks.mock(ICaseService.class);
        
        // Create action and inject mock
        TA_Case_WithDependencies action = new TA_Case_WithDependencies();
        action.caseService = mockService;
        
        // Create trigger context
        Case oldCase = new Case(Id = fflib_IDGenerator.generate(Case.SObjectType));
        Case newCase = oldCase.clone(true);
        
        fflib_TriggerContext context = new fflib_TriggerContext();
        context.newMap = new Map<Id, Case>{newCase.Id => newCase};
        context.oldMap = new Map<Id, Case>{oldCase.Id => oldCase};
        
        action.triggerContext = context;
        
        Test.startTest();
        action.onAfterUpdate();
        Test.stopTest();
        
        // Verify service was called
        ((ICaseService) mocks.verify(mockService, 1))
            .processUpdatedCases(context.newMap.keySet());
    }
}
```

## Migration Strategy

### From Monolithic to Modular

1. **Analyze Existing Handler**
   - List all operations in current handler
   - Group related operations
   - Identify dependencies

2. **Create Trigger Actions**
   - One action per logical operation
   - Keep actions under 50 lines
   - Single responsibility principle

3. **Configure Metadata**
   - Create custom metadata records
   - Set appropriate sequence
   - Configure trigger contexts

4. **Gradual Migration**
   ```apex
   // Transitional approach - both patterns
   trigger AccountTrigger on Account (...) {
       // New pattern
       new fflib_TriggerHandler().run();
       
       // Old pattern (temporarily)
       AccountTriggerHandler.handle();
   }
   ```

5. **Testing**
   - Test each action individually
   - Integration test full flow
   - Performance test at scale

## Benefits of Trigger Actions

### Development Benefits
- **Focused Classes**: Each action does one thing
- **Parallel Development**: No merge conflicts
- **Easy Testing**: Test individual actions
- **Reusability**: Actions can be shared across objects

### Operational Benefits
- **Configuration**: Enable/disable without deployment
- **Sequencing**: Control execution order via metadata
- **Debugging**: Isolate issues to specific actions
- **Performance**: Only run necessary actions

### Maintenance Benefits
- **Clear Responsibilities**: Easy to understand
- **Version Control**: Better diff visibility
- **Code Reviews**: Review small changes
- **Documentation**: Self-documenting actions

## Best Practices

### 1. Naming Conventions
```
TA_{Object}_{Action}
Examples:
- TA_Case_SendEscalationEmails
- TA_Account_ValidateAddress
- TA_Opportunity_CalculateScore
```

### 2. Action Granularity
- One business operation per action
- Keep under 50 lines of code
- Single trigger context per action

### 3. Metadata Organization
```xml
<CustomMetadata>
    <label>{Object} - {Description}</label>
    <values>
        <field>ObjectType__c</field>
        <value>{SObject API Name}</value>
    </values>
    <values>
        <field>ImplementationType__c</field>
        <value>{Class Name}</value>
    </values>
    <values>
        <field>Sequence__c</field>
        <value>{10, 20, 30...}</value> <!-- Leave gaps -->
    </values>
</CustomMetadata>
```

### 4. Error Handling
```apex
public override void onAfterInsert() {
    try {
        // Action logic
    } catch (Exception e) {
        // Log error but don't fail transaction
        Logger.error('Failed to execute action', e);
        Logger.saveLog();
    }
}
```

## Common Patterns

### Validation Actions
```apex
public class TA_Object_Validate extends fflib_TriggerAction {
    public override void onBeforeInsert() {
        for (SObject record : triggerContext.newList) {
            if (!isValid(record)) {
                record.addError('Validation failed');
            }
        }
    }
}
```

### Field Update Actions
```apex
public class TA_Object_UpdateFields extends fflib_TriggerAction {
    public override void onBeforeUpdate() {
        for (SObject record : triggerContext.newList) {
            SObject oldRecord = triggerContext.oldMap.get(record.Id);
            if (hasRelevantChange(record, oldRecord)) {
                updateDerivedFields(record);
            }
        }
    }
}
```

### Integration Actions
```apex
public class TA_Object_PublishEvent extends fflib_TriggerAction {
    public override void onAfterInsert() {
        List<Platform_Event__e> events = new List<Platform_Event__e>();
        for (SObject record : triggerContext.newList) {
            events.add(createEvent(record));
        }
        EventBus.publish(events);
    }
}
```

## Monitoring and Debugging

### Custom Metadata Dashboard
Create reports/dashboards to visualize:
- Active trigger actions per object
- Execution sequence
- Recently modified actions

### Production Logging with Nebula Logger
```apex
public override void onAfterUpdate() {
    // Use Nebula Logger for production monitoring
    Logger.info('Starting escalation email processing')
        .setField('recordCount', triggerContext.newList.size())
        .setField('triggerAction', 'TA_Case_SendEscalationEmails');
    
    try {
        // Action logic
        processEscalations();
        
        Logger.info('Completed escalation email processing');
    } catch (Exception e) {
        Logger.error('Failed to process escalations', e);
    } finally {
        Logger.saveLog();
    }
}

// For development/debugging only (remove before deployment)
private void temporaryDebugLogging() {
    // System.debug should only be used during development
    // and must be removed before production deployment
    System.debug('Temporary debug for investigation');
}
```

## Conclusion

The Trigger Actions pattern transforms unmaintainable monolithic handlers into:
- **Modular** single-purpose actions
- **Configurable** metadata-driven execution
- **Testable** isolated units
- **Maintainable** focused classes

This approach has proven successful in production systems with hundreds of trigger actions across dozens of objects, enabling teams to work in parallel without conflicts while maintaining code quality.