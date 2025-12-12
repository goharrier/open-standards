# Domain Layer Pattern with Fluent Interface

### **Overview**

{% hint style="info" %}
**When to use this pattern:** When building business logic that operates on collections of SObjects, requires filtering/transformation operations, and needs to maintain readability while ensuring testability and reusability.
{% endhint %}

#### **Purpose**

The Domain Layer with Fluent Interface pattern encapsulates business logic for SObject types in dedicated classes that support method chaining. This enables expressive, readable code that clearly communicates intent while providing strong type safety and comprehensive testability.

#### **Context**

Enterprise Salesforce implementations often suffer from:

* Business logic scattered across triggers, services, and controllers
* Repeated filtering and transformation code
* Difficult-to-test code due to tight coupling with SObjects
* Poor readability due to verbose iteration patterns

The Domain Layer consolidates all SObject-specific logic into cohesive, chainable interfaces.

### **Problem Statement**

#### **The Challenge**

Working directly with `List<SObject>` leads to several issues:

```
// Anti-pattern: Verbose, repetitive, hard to maintain
List<Account> activeAccounts = new List<Account>();
for (Account acc : accounts) {
    if (acc.Account_Status__c == 'Active') {
        activeAccounts.add(acc);
    }
}

List<Account> activeUsAccounts = new List<Account>();
for (Account acc : activeAccounts) {
    if (acc.BillingCountryCode == 'US') {
        activeUsAccounts.add(acc);
    }
}

for (Account acc : activeUsAccounts) {
    acc.Invoice_Batch__c = 'Batch 1';
}

```

This pattern repeats throughout the codebase, is error-prone, and obscures business intent.

#### **Why Traditional Approaches Fall Short**

* **Utility Classes**: Static methods don't support chaining and feel disconnected from the data
* **Extension Methods**: Apex doesn't support them
* **Direct List Manipulation**: No encapsulation, duplicated filtering logic, poor testability

### **Solution**

#### **Core Concept**

Wrap `List<SObject>` in domain classes that expose business operations as chainable methods. Selection methods (`select*`) return filtered domain instances, mutation methods (`set*`) modify records and return the same instance for chaining and accessor methods (`get*`) retrieve data from the records (primitive data types).

#### **Implementation Strategy**

#### **1. Define the Domain Interface**

The interface declares all operations available on the domain:

```
public interface IAccounts extends fflib_ISObjects {
    // Accessor methods - extract data
    List<Account> getAccounts();
    Map<String, Id> getAccountIdByUmId();
    Set<String> getUmIds();
   
    // Selection methods - return filtered domain
    IAccounts selectBlankBillingAddress();
    IAccounts selectByBillingCountryCode(String countryCode);
    IAccounts selectByBillingCountryCode(Set<String> countryCodes);
    IAccounts selectByPaymentMethod(Set<String> paymentMethods);
    IAccounts selectByStatus(String statusValue);
    IAccounts selectByStatusIn(Set<String> statusValues);
    IAccounts selectWithNetsuiteId();

    // Mutation methods - modify and return self
    IAccounts copyBillingCountryCodeToShippingCountryCode();
    IAccounts copyShippingAddressToBillingAddress();
    IAccounts setAccountStatus(String accountStatus);
    IAccounts setInvoiceBatch(String invoiceBatch);
    IAccounts setPaymentTerm(String paymentTerm);
}
```

#### **2. Implement the Domain Class**

The domain class extends the framework base and implements the interface:

```
public virtual inherited sharing class Accounts extends fflib_SObjects2 implements IAccounts {

    // Static factory methods for clean instantiation
    public static IAccounts newInstance(List<Account> accounts) {
        return (IAccounts) Application.domain.newInstance(accounts, Schema.Account.SObjectType);
    }

    public static IAccounts newInstance(Set<Id> accountIds) {
        return (IAccounts) Application.domain.newInstance(accountIds);
    }

    // Constructor
    public Accounts(List<Account> accounts) {
        super(accounts, Schema.Account.SObjectType);
    }
    
    
    // ============================================
    // ACCESSOR METHODS - Extract data
    // ============================================

    public List<Account> getAccounts() {
        return getRecords();
    }

    public Set<String> getUmIds() {
        return getStringFieldValues(Account.UM_Id__c);
    }

    public Map<String, Id> getAccountByUmId() {
        return getIdFieldByStringField(
                Schema.Account.Id,
                Schema.Account.Account.UM_Id__c);
    }

    // ============================================
    // SELECTION METHODS - Return filtered domain
    // ============================================

    public IAccounts selectBlankBillingAddress() {
        List<Account> result = new List<Account>();
        for (Account record : getAccounts()) {
            if (String.isBlank(record.BillingStreet)
                    && String.isBlank(record.BillingCity)
                    && String.isBlank(record.BillingCountry)
                    && String.isBlank(record.BillingPostalCode)
            ) {
                result.add(record);
            }
        }
        return new Accounts(result);
    }
    
    public IAccounts selectByBillingCountryCode(String countryCode) {
        return selectByBillingCountryCode(new Set<String>{countryCode});
    }

    public IAccounts selectByBillingCountryCode(Set<String> countryCodes) {
        return new Accounts(getRecords(Schema.Account.BillingCountryCode, countryCodes));
    }
    
    public IAccounts selectByStatus(String statusValue) {
        return selectByStatusIn(new Set<String>{statusValue});
    }

    public IAccounts selectByStatusIn(Set<String> statusValues) {
        return new Accounts(getRecords(Schema.Account.Account_Status__c, statusValues));
    }

    public IAccounts selectByPaymentMethod(Set<String> paymentMethods) {
        return new Accounts(getRecords(Schema.Account.Payment_Method__c, paymentMethods));
    }    

    public IAccounts selectWithNetsuiteId() {
        return new Accounts(selectNonBlank(Schema.Account.Netsuite_Id__c));
    }

    // ============================================
    // MUTATION METHODS - Modify and return self
    // ============================================
    public IAccounts copyShippingAddressToBillingAddress() {
        for (Account record : getAccounts()) {
            record.BillingCity = record.ShippingCity;
            record.BillingCountryCode = record.ShippingCountryCode;
            record.BillingPostalCode = record.ShippingPostalCode;
            record.BillingStreet = record.ShippingStreet;
            record.BillingStateCode = record.ShippingStateCode;
        }
        return this;
    }
    
    public IAccounts setAccountStatus(String accountStatus) {
        setFieldValue(Schema.Account.Account_Status__c, accountStatus);
        return this;
    }

    public IAccounts setInvoiceBatch(String invoiceBatch) {
        setFieldValue(Account.Invoice_Batch__c, invoiceBatch);
        return this;
    }

    public IAccounts copyBillingCountryCodeToShippingCountryCode() {
        for (Account record : getAccounts()) {
            record.ShippingCountryCode = record.BillingCountryCode;
        }
        return this;
    }
    
    public IAccounts setPaymentTerm(String paymentTerm) {
        setFieldValue(Schema.Account.Payment_Term__c, paymentTerm);
        return this;
    }

    // ============================================
    // PRIVATE HELPERS
    // ============================================
    
	  @TestVisible
	  protected List<Account> selectWith(Schema.SObjectField sObjectField) {
		    List<Account> result = new List<Account>();
		    for (Account record : getAccounts()) {
    			  if (record.get(sObjectField) == null) continue;

		    	  result.add(record);
		    }
		    return result;
	  }
	  
	  @TestVisible
	  protected List<Account> selectWithout(Schema.SObjectField sObjectField) {
		    List<Account> result = new List<Account>();
		    for (Account record : getAccounts()) {
			      if (record.get(sObjectField) != null) continue;

	      		result.add(record);
		    }
		    return result;
	  }	  

    // ============================================
    // DOMAIN CONSTRUCTOR (for Application factory)
    // ============================================

    public class Constructor implements fflib_IDomainConstructor {
        public fflib_SObjects construct(List<Object> accounts) {
            return new Accounts((List<Account>) accounts);
        }
    }
}

```

#### **Key Components**

| Component            | Purpose        | Responsibility                                   |
| -------------------- | -------------- | ------------------------------------------------ |
| `IAccounts`          | Interface      | Defines the public API for the domain            |
| `Accounts`           | Implementation | Contains all business logic for Account records  |
| `fflib_SObjects2`    | Base Class     | Provides reusable selection and accessor methods |
| `Application.domain` | Factory        | Creates domain instances, supports mocking       |
| `Constructor`        | Inner Class    | Enables factory-based instantiation              |

### **Implementation Details**

#### **Required Setup**

1. **Framework Dependency**: Install `fflib-apex-common` and `fflib-apex-extensions`
2. **Application Class**: Configure domain factory:

```
public class Application {
    public static final fflib_Application.DomainFactory domain =
        new fflib_Application.DomainFactory(
            selector,
            new Map<SObjectType, Type>{
                Account.SObjectType => Accounts.Constructor.class
            }
        );
}

```

#### **Code Structure - Using the Domain in Trigger Handler**

```
public with sharing class AccountTriggerHandler extends fflib_SObjectDomain {

    public override void onBeforeInsert() {
        IAccounts accounts = Accounts.newInstance(getRecords());

        // Fluent chaining makes business logic readable
        accounts
            .selectBlankBillingAddress()
            .copyShippingAddressToBillingAddress();

        accounts
            .selectWithEmptyShippingCountryCode()
            .copyBillingCountryCodeToShippingCountryCode();

        setPaymentTerm(accounts);
        setInvoiceBatch(accounts);
    }

    /**
     * Sets the payment term based on payment method.
     * Recurring methods get 'Due on Receipt', others get 'Net 30'.
     */
    private static void setPaymentTerm(IAccounts accounts) {
        Set<String> recurringPaymentMethods = new Set<String>{
            'Recurring Credit Card', 'Direct Debit', 'ACH'
        };
        accounts
            .selectByPaymentMethod(recurringPaymentMethods)
            .setPaymentTerm('Due on Receipt');

        Set<String> nonRecurringPaymentMethods = new Set<String>{
            'Credit Card', 'Wire Transfer', 'Boletor', '', null
        };
        accounts
            .selectByPaymentMethod(nonRecurringPaymentMethods)
            .selectByPaymentTerm(new Set<String>{'Due on receipt', '', null})
            .setPaymentTerm('Net 30');
    }

    /**
     * Sets invoice batch based on country, payment method, and billing period.
     * Complex business rules expressed clearly through chaining.
     */
    private static void setInvoiceBatch(IAccounts domain) {
        // Skip accounts with protected batch assignments
        IAccounts accounts = domain.selectByInvoiceBatchNotIn(
            new Set<String>{'Batch 2', 'Batch 3', 'Batch 8', 'Batch 13'}
        );
        if (accounts.isEmpty()) return;

        // Brazil credit card/wire/boletos -> Batch 6
        Set<Id> processedIds = accounts
            .selectByBillingCountry('Brazil')
            .selectByPaymentMethod(new Set<String>{'Credit Card', 'Wire Transfer', 'Boletos'})
            .setInvoiceBatch('Batch 6')
            .getRecordIds();

        accounts = accounts.selectByIdNotIn(processedIds);
        if (accounts.isEmpty()) return;

        // Direct Debit/ACH -> Batch 4
        processedIds = accounts
            .selectByPaymentMethod(new Set<String>{'Direct Debit', 'ACH'})
            .setInvoiceBatch('Batch 4')
            .getRecordIds();

        accounts = accounts.selectByIdNotIn(processedIds);
        if (accounts.isEmpty()) return;

        // Monthly/Quarterly billing -> Batch 7
        processedIds = accounts
            .selectByBillingPeriod(new Set<String>{'Monthly', 'Quarterly'})
            .setInvoiceBatch('Batch 7')
            .getRecordIds();

        accounts = accounts.selectByIdNotIn(processedIds);
        if (accounts.isEmpty()) return;

        // Default -> Batch 1
        accounts.setInvoiceBatch('Batch 1');
    }
}

```

#### **Configuration Requirements**

* **Custom Settings/Metadata**: None required for the pattern itself
* **Permissions**: Standard object permissions apply
* **Dependencies**: `fflib-apex-common`, `fflib-apex-extensions`

### **Best Practices**

#### **Do's**

* Name selection methods with `selectBy*` or `selectWith*` prefix
* Name mutation methods with `set*` prefix
* Return `this` from mutation methods to enable chaining
* Return new domain instances from selection methods (immutable filtering)
* Use Schema tokens (`Schema.Account.Field__c`) for compile-time safety
* Provide both single-value and set-based overloads for flexibility
* Include `isEmpty()` checks to short-circuit empty collections

#### **Don'ts**

* Do not perform DML inside domain methods (that belongs in Services or UoW)
* Do not query data inside domain methods (domains work on in-memory records)
* Do not throw exceptions from selection methods (return empty domain instead)
* Do not mix selection and mutation in a single method

### **Considerations**

#### **Governor Limits**

* Domain operations are in-memory, no SOQL/DML overhead
* Chained operations iterate the collection once per method (be mindful with large datasets)
* Early `isEmpty()` returns prevent unnecessary iterations

#### **Performance Impact**

* In-memory filtering is extremely fast
* Each selection creates a new List, but this is negligible for typical record volumes
* For very large datasets (10k+ records), consider batch processing instead

#### **Security Implications**

* Use `with sharing` on domain classes to respect record-level security
* Field-level security is not automatically enforced; consider using `Security.stripInaccessible()`

### **Variations**

#### **Variation 1: Exclusion Methods**

Add `selectByFieldNotIn` methods for inverse filtering:

```
public IAccounts selectByStatusNotIn(Set<String> statusValues) {
    return new Accounts(getRecordsNotIn(Schema.Account.Account_Status__c, statusValues));
}

public IAccounts selectByIdNotIn(Set<Id> accountIds) {
    return new Accounts(getRecordsNotIn(Schema.Account.Id, accountIds));
}

```

#### **Variation 2: Conditional Mutation**

Combine selection with map-based value assignment:

```
public IAccounts setNumberOfHotelsById(Map<Id, Decimal> countByAccountIds) {
    setFieldValue(Account.Id, Account.Number_of_Hotels_Management__c, countByAccountIds);
    return this;
}

public IAccounts setPreviousAccountOwner(Map<Id, Id> ownerIdByAccountId) {
    setFieldValue(Schema.Account.Id, Schema.Account.Previous_Account_Owner__c, ownerIdByAccountId);
    return this;
}

```

#### **Variation 3: Cross-Domain Data Extraction**

Extract data for use with related domains:

```
public Set<Id> getManagementCompanyIds() {
    return getIdFieldValues(Schema.Account.Management_Company__c);
}

public Map<Id, Id> getOwnerIdById() {
    return getIdFieldByIdField(Schema.Account.OwnerId, Schema.Account.Id);
}

public Map<String, Set<Id>> getIdsByInvoiceBatch() {
    return getIdFieldsByStringField(Schema.Account.Id, Schema.Account.Invoice_Batch__c);
}

```

### **Testing Approach**

#### **Unit Test Strategy**

```
@IsTest(IsParallel=true)
private class AccountsTest {

    @IsTest
    static void itShouldSelectByStatus() {
        // GIVEN accounts with different statuses
        List<Account> records = new List<Account>{
            new Account(Account_Status__c = 'Active'),
            new Account(Account_Status__c = 'Inactive'),
            new Account(Account_Status__c = 'Active')
        };

        // WHEN we select by status
        IAccounts result = new Accounts(records).selectByStatus('Active');

        // THEN only active accounts are returned
        System.Assert.areEqual(2, result.getAccounts().size());
        for (Account acc : result.getAccounts()) {
            System.Assert.areEqual('Active', acc.Account_Status__c);
        }
    }

    @IsTest
    static void itShouldChainSelectionsAndMutations() {
        // GIVEN accounts with various attributes
        List<Account> records = new List<Account>{
            new Account(Account_Status__c = 'Active', BillingCountryCode = 'US'),
            new Account(Account_Status__c = 'Active', BillingCountryCode = 'NL'),
            new Account(Account_Status__c = 'Inactive', BillingCountryCode = 'US')
        };

        // WHEN we chain selections and set invoice batch
        new Accounts(records)
            .selectByStatus('Active')
            .selectByBillingCountryCode('US')
            .setInvoiceBatch('Batch 1');

        // THEN only the matching record is updated
        System.Assert.areEqual('Batch 1', records[0].Invoice_Batch__c);
        System.Assert.isNull(records[1].Invoice_Batch__c);
        System.Assert.isNull(records[2].Invoice_Batch__c);
    }

    @IsTest
    static void itShouldCopyAddressFields() {
        // GIVEN account with shipping address but no billing
        Account record = new Account(
            ShippingStreet = '123 Main St',
            ShippingCity = 'Amsterdam',
            ShippingCountryCode = 'NL'
        );

        // WHEN we copy shipping to billing
        new Accounts(new List<Account>{record})
            .copyShippingAddressToBillingAddress();

        // THEN billing address matches shipping
        System.Assert.areEqual('123 Main St', record.BillingStreet);
        System.Assert.areEqual('Amsterdam', record.BillingCity);
        System.Assert.areEqual('NL', record.BillingCountryCode);
    }
}

```

#### **Test Scenarios**

1. **Selection Filtering**: Each `selectBy*` method correctly filters records
2. **Method Chaining**: Chained operations work correctly together
3. **Mutation**: `set*` methods correctly update field values
4. **Empty Collections**: Methods handle empty collections gracefully
5. **Accessor Methods**: Data extraction methods return correct values

### **Trade-offs**

| Benefit                                   | Cost                                    |
| ----------------------------------------- | --------------------------------------- |
| Highly readable, expressive code          | Learning curve for team members         |
| Excellent testability (no mocking needed) | Additional classes to maintain          |
| Strong type safety via interfaces         | Memory overhead from creating new Lists |
| Reusable across triggers, services, batch | Framework dependency                    |
| Self-documenting business logic           | Upfront implementation effort           |
