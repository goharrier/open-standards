# JSON Field Storage Pattern for Non-Transactional Data

## Problem Context

In Salesforce development, we often encounter scenarios where we need to store structured, supplementary data that:
- Is primarily for display or informational purposes
- Won't be used in SOQL queries, workflows, or process automation
- Varies in structure between records
- Would otherwise require creating numerous custom fields or related objects
- Needs to maintain flexibility for future changes without schema modifications

Traditional approaches like creating custom objects or fields for every piece of data can lead to:
- Object proliferation and complexity
- Hitting Salesforce limits (custom field limits, object limits)
- Maintenance overhead for rarely-used fields
- Performance impacts from excessive joins
- Schema rigidity that hampers rapid iteration

## Solution: JSON Storage in LongTextArea Fields

The pattern leverages Salesforce's `LongTextArea` fields to store JSON-serialized data, providing a flexible, schema-less approach for non-transactional information.

### Field Definition Pattern

Define a generic `Metadata__c` field on objects that need flexible data storage:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<CustomField xmlns="http://soap.sforce.com/2006/04/metadata">
    <fullName>Metadata__c</fullName>
    <description>Stores supplementary data as JSON for display purposes</description>
    <externalId>false</externalId>
    <label>Metadata</label>
    <length>131072</length>
    <trackHistory>false</trackHistory>
    <trackTrending>false</trackTrending>
    <type>LongTextArea</type>
    <visibleLines>3</visibleLines>
</CustomField>
```

**Key Configuration Decisions:**
- **Field Type**: `LongTextArea` provides up to 131,072 characters (sufficient for complex JSON structures)
- **History Tracking**: Generally disabled for performance (JSON diffs are not user-friendly)
- **Field Naming**: Use consistent naming like `Metadata__c` or domain-specific names like `ReviewData__c`

## Implementation Patterns

### 1. Configuration Storage Pattern

Store variable configuration mappings that drive application behavior:

```apex
public class DamageQuestionnaireConfigParser {
    public Map<String, Id> parse(TortConfiguration__c record) {
        try {
            // Parse JSON configuration mapping damage types to questionnaire IDs
            return (Map<String, Id>) JSON.deserialize(
                record.Metadata__c, 
                Map<String, Id>.class
            );
        } catch (Exception e) {
            Logger.error('TortConfiguration__c.Metadata__c parsing failed.', record, e);
            return null;
        }
    }
}
```

**Example JSON Structure:**
```json
{
    "Real Property": "a0X1234567890ABC",
    "Personal Property": "a0X1234567890DEF",
    "Business Loss": "a0X1234567890GHI"
}
```

### 2. Dynamic Options Storage Pattern

Store dynamic form options that vary by record:

```apex
public class QuestionOptionParser {
    public List<QuestionOption> parse(DamageQuestion__c record) {
        if (!isSelectType(record)) return null;
        
        try {
            return (List<QuestionOption>) JSON.deserialize(
                record.Options__c, 
                List<QuestionOption>.class
            );
        } catch (Exception e) {
            Logger.error('DamageQuestion__c.Options__c parsing failed.', record, e);
            return new List<QuestionOption>();
        }
    }
}
```

**Example JSON Structure:**
```json
[
    {"label": "Single Family Home", "value": "single_family"},
    {"label": "Multi-Family", "value": "multi_family"},
    {"label": "Commercial", "value": "commercial"}
]
```

### 3. Document Metadata Pattern

Store supplementary document information from external systems:

```apex
public class FileUploadController {
    @AuraEnabled
    public static String createDocument(DocumentInfo documentInfo) {
        Document__c document = new Document__c(
            Case__c = documentInfo.caseId,
            FileName__c = documentInfo.fileName,
            FilePath__c = documentInfo.filePath,
            Metadata__c = documentInfo.metadata,  // JSON from external system
            Type__c = documentInfo.type
        );
        
        // Use Unit of Work pattern for transaction management
        fflib_ISObjectUnitOfWork unitOfWork = Application.UnitOfWork.newInstance(
            new List<SObjectType>{ Document__c.SObjectType }
        );
        
        unitOfWork.registerNew(document);
        unitOfWork.commitWork();
        return document.Id;
    }
}
```

## Display Pattern in Lightning Web Components

### Parsing and Flattening JSON for Display

The `metadataView` LWC demonstrates how to parse and display JSON data:

```javascript
export default class MetadataView extends LightningElement {
    metadata = [];
    
    handleLightningEvent(payload) {
        try {
            const parsedMetadata = JSON.parse(payload.body.metadata);
            this.metadata = this.flattenMetadata(parsedMetadata);
        } catch (error) {
            this.metadata = [];
        }
    }
    
    flattenMetadata(metadata) {
        // Recursive flattening for nested JSON structures
        function flattenObject(objectValue) {
            return Object.keys(objectValue).reduce((result, key) => {
                const value = objectValue[key];
                const defaultValue = { label: key, value };
                
                // Handle nested objects/arrays recursively
                if (typeof value === 'object' && value !== null) {
                    return result.concat(flattenObject(value));
                }
                
                return result.concat(defaultValue);
            }, []);
        }
        
        return flattenObject(metadata);
    }
}
```

### Display Template Pattern

```html
<template lwc:if={metadata.length}>
    <lightning-layout multiple-rows>
        <template for:each={metadata} for:item="field">
            <lightning-layout-item key={field.label} padding="horizontal-small" size="6">
                <div class="slds-form-element">
                    <span class="slds-form-element__label">{field.label}</span>
                    <div class="slds-form-element__static">{field.value}</div>
                </div>
            </lightning-layout-item>
        </template>
    </lightning-layout>
</template>
```

## Benefits of This Approach

### 1. **Schema Flexibility**
- Add new data attributes without deploying metadata changes
- Adapt to changing requirements without modifying object schema
- Support varying data structures across records of the same type

### 2. **Performance Optimization**
- Reduce custom field count (helps with query selectivity)
- Minimize object relationships for non-queryable data
- Single field retrieval for all supplementary data

### 3. **Development Velocity**
- Rapid iteration on data structures
- No deployment dependencies for data model changes
- Easier integration with external systems that provide JSON

### 4. **Maintenance Simplicity**
- Centralized storage pattern
- Consistent parsing approach
- Clear separation between transactional and display data

## When to Use This Pattern

### Ideal Use Cases
- **External System Data**: Information from APIs that won't be queried
- **Configuration Storage**: Variable settings that differ by context
- **Document Metadata**: File properties, processing results, audit information
- **Form Responses**: Dynamic questionnaire answers, survey data
- **Integration Payloads**: Preserving original data from external sources
- **Audit/History Data**: Snapshots of record states for reference

### When NOT to Use This Pattern
- **Queryable Data**: Any field that needs SOQL filtering
- **Workflow Criteria**: Data used in automation rules
- **Reporting Fields**: Information needed in reports/dashboards
- **Frequently Updated Data**: High-volume transactional updates
- **Regulated Data**: Information requiring field-level security

## Best Practices and Considerations

### 1. **Error Handling**
Always implement robust error handling for JSON parsing:

```apex
try {
    return JSON.deserialize(jsonString, TargetType.class);
} catch (Exception e) {
    // Log error with context
    Logger.error('JSON parsing failed for record', record, e);
    // Return safe default
    return getDefaultValue();
}
```

### 2. **Validation**
Consider implementing JSON schema validation for critical data:

```apex
public Boolean isValidMetadata(String jsonString) {
    try {
        Map<String, Object> parsed = (Map<String, Object>) JSON.deserializeUntyped(jsonString);
        // Validate required keys exist
        return parsed.containsKey('requiredField1') 
            && parsed.containsKey('requiredField2');
    } catch (Exception e) {
        return false;
    }
}
```

### 3. **Size Management**
Monitor field usage to avoid hitting the 131KB limit:

```apex
public void saveMetadata(String jsonData) {
    if (jsonData.length() > 130000) {
        throw new MetadataException('Metadata exceeds size limit');
    }
    // Proceed with save
}
```

### 4. **Documentation**
Always document the expected JSON structure:

```apex
/**
 * Metadata__c field structure:
 * {
 *   "source": "ExternalSystem",
 *   "processedDate": "2024-01-15",
 *   "attributes": {
 *     "category": "TypeA",
 *     "priority": "High"
 *   }
 * }
 */
```

### 5. **Type Safety**
Create wrapper classes for complex structures:

```apex
public class DocumentMetadata {
    public String source;
    public DateTime processedDate;
    public Map<String, String> attributes;
    
    public static DocumentMetadata parse(String jsonString) {
        return (DocumentMetadata) JSON.deserialize(
            jsonString, 
            DocumentMetadata.class
        );
    }
}
```

## Common Pitfalls to Avoid

1. **Using JSON fields for queryable data** - This defeats the purpose and creates maintenance nightmares
2. **Storing sensitive data** - JSON fields bypass field-level security
3. **Neglecting error handling** - Invalid JSON will break your application
4. **Over-nesting structures** - Keep JSON reasonably flat for maintainability
5. **Missing data migration strategy** - Plan for structure evolution
6. **Ignoring governor limits** - Large JSON processing can hit CPU limits

## Migration and Evolution Strategy

When JSON structures need to evolve:

```apex
public class MetadataVersionManager {
    public Map<String, Object> migrate(String jsonString) {
        Map<String, Object> data = (Map<String, Object>) JSON.deserializeUntyped(jsonString);
        
        // Check version
        Integer version = (Integer) data.get('version');
        if (version == null) version = 1;
        
        // Apply migrations sequentially
        if (version < 2) {
            data = migrateV1ToV2(data);
        }
        if (version < 3) {
            data = migrateV2ToV3(data);
        }
        
        return data;
    }
}
```

## Conclusion

The JSON field storage pattern provides a powerful solution for managing non-transactional, display-oriented data in Salesforce. By understanding when and how to apply this pattern, development teams can build more flexible, maintainable applications while avoiding the overhead of excessive custom fields and objects.

Remember: This pattern complements, not replaces, proper data modeling. Use it judiciously for the right use cases to maximize its benefits while maintaining system integrity and performance.