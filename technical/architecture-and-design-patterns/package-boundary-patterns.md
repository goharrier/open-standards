# Package Boundary Patterns

## Introduction

Defining clear package boundaries is crucial for maintainable, scalable Salesforce solutions. This document presents patterns for establishing and maintaining package boundaries based on production implementations managing 40+ packages.

## Boundary Definition Strategies

### 1. Domain-Driven Boundaries

Packages align with business domains, containing all layers for that domain.

```
package-structure/
├── customer-management/
│   ├── domain/          # Customer, Contact domain logic
│   ├── services/        # Customer-specific services
│   ├── selectors/       # Customer data access
│   └── ui/             # Customer-specific UI
├── order-management/
│   ├── domain/          # Order, OrderItem logic
│   ├── services/        # Order processing
│   ├── selectors/       # Order data access
│   └── ui/             # Order UI components
└── inventory/
    ├── domain/          # Product, Stock logic
    ├── services/        # Inventory services
    ├── selectors/       # Inventory queries
    └── ui/             # Inventory UI
```

**When to Use:**
- Clear business domain separation
- Different teams own different domains
- Domains have different change frequencies

### 2. Layer-Based Boundaries

Packages organized by architectural layers, cutting across domains.

```
package-structure/
├── common-ui/           # Shared UI components
├── common-services/     # Shared services
├── common-domain/       # Shared domain logic
├── common-data/         # Shared data access
└── common-integration/  # Shared integrations
```

**When to Use:**
- High reuse across domains
- Consistent technical patterns
- Small team maintaining all domains

### 3. Capability-Based Boundaries

Packages provide specific capabilities independent of domain.

```
package-structure/
├── authentication/      # Auth capability
├── document-generation/ # Document capability
├── notification/       # Notification capability
├── analytics/          # Analytics capability
└── workflow/           # Workflow capability
```

**When to Use:**
- Cross-cutting concerns
- Reusable across multiple orgs
- Third-party integrations

## Boundary Enforcement Patterns

### 1. Interface Segregation Pattern

Each package exposes minimal, focused interfaces.

```apex
// ❌ BAD: Fat interface
public interface ICustomerService {
    Customer create(CustomerRequest req);
    void update(Customer c);
    void delete(Id customerId);
    List<Customer> search(String criteria);
    void merge(Id primary, Id secondary);
    void validateAddress(Address addr);
    CreditScore checkCredit(Id customerId);
    List<Order> getOrders(Id customerId);
    // ... 20 more methods
}

// ✅ GOOD: Segregated interfaces
public interface ICustomerCommandService {
    Customer create(CustomerRequest req);
    void update(Customer c);
    void delete(Id customerId);
}

public interface ICustomerQueryService {
    Customer getById(Id customerId);
    List<Customer> search(SearchCriteria criteria);
}

public interface ICustomerValidationService {
    ValidationResult validateCustomer(Customer c);
    AddressValidation validateAddress(Address addr);
}
```

### 2. Dependency Injection Pattern

Packages declare dependencies explicitly through constructor injection or property injection.

```apex
// Package service with explicit dependencies
public class OrderService implements IOrderService {
    private final ICustomerQueryService customerService;
    private final IInventoryService inventoryService;
    private final IPricingService pricingService;
    
    // Constructor injection
    public OrderService() {
        this(
            (ICustomerQueryService) Application.Service.newInstance(ICustomerQueryService.class),
            (IInventoryService) Application.Service.newInstance(IInventoryService.class),
            (IPricingService) Application.Service.newInstance(IPricingService.class)
        );
    }
    
    @TestVisible
    private OrderService(
        ICustomerQueryService customerService,
        IInventoryService inventoryService,
        IPricingService pricingService
    ) {
        this.customerService = customerService;
        this.inventoryService = inventoryService;
        this.pricingService = pricingService;
    }
}
```

### 3. Package Registry Pattern

Central registry for package capabilities and services.

```apex
// Central package registry
public class PackageRegistry {
    private static Map<String, PackageInfo> packages = new Map<String, PackageInfo>();
    
    public class PackageInfo {
        public String name;
        public String version;
        public Map<Type, Type> services;
        public Set<String> capabilities;
    }
    
    // Package self-registration
    static {
        registerPackage('customer-management', new PackageInfo()
            .withService(ICustomerService.class, CustomerServiceImpl.class)
            .withCapability('CUSTOMER_CRUD')
            .withCapability('CUSTOMER_SEARCH')
        );
    }
    
    public static Object getService(Type serviceInterface) {
        for (PackageInfo pkg : packages.values()) {
            if (pkg.services.containsKey(serviceInterface)) {
                return pkg.services.get(serviceInterface).newInstance();
            }
        }
        return null;
    }
}
```

## Cross-Package Communication Patterns

### 1. Service Mesh Pattern

Packages communicate through a service mesh layer that handles routing, security, and monitoring.

```apex
// Service mesh router
public class ServiceMesh {
    public static ServiceResponse invoke(ServiceRequest request) {
        // Routing logic
        String targetPackage = resolvePackage(request.service);
        
        // Security check
        validateAccess(request.caller, targetPackage);
        
        // Monitoring
        Long startTime = System.currentTimeMillis();
        
        try {
            // Invoke service
            Object service = PackageRegistry.getService(request.service);
            Object result = invokeMethod(service, request.method, request.params);
            
            // Log metrics
            logMetrics(request, System.currentTimeMillis() - startTime);
            
            return new ServiceResponse(result);
        } catch (Exception e) {
            handleError(request, e);
            throw e;
        }
    }
}
```

### 2. Event-Driven Communication

Packages communicate through Platform Events for loose coupling.

```apex
// Publishing package
public class OrderService {
    public void completeOrder(Id orderId) {
        // Business logic
        processOrder(orderId);
        
        // Publish event for other packages
        EventBus.publish(new OrderCompleted__e(
            OrderId__c = orderId,
            CompletedDate__c = DateTime.now(),
            TotalAmount__c = calculateTotal(orderId)
        ));
    }
}

// Subscribing package
public class FulfillmentService {
    // Platform Event trigger
    public static void handleOrderCompleted(List<OrderCompleted__e> events) {
        for (OrderCompleted__e event : events) {
            createFulfillment(event.OrderId__c);
        }
    }
}
```

### 3. Configuration-Driven Routing

Use Custom Metadata to configure package interactions.

```apex
// Custom Metadata: PackageRoute__mdt
// ServiceInterface__c: 'ICustomerService'
// Implementation__c: 'CustomerServiceImpl'
// Package__c: 'customer-management'
// IsActive__c: true

public class DynamicServiceFactory {
    public static Object createService(Type serviceInterface) {
        PackageRoute__mdt route = [
            SELECT Implementation__c, Package__c
            FROM PackageRoute__mdt
            WHERE ServiceInterface__c = :serviceInterface.getName()
                AND IsActive__c = true
            LIMIT 1
        ];
        
        if (route != null) {
            Type implType = Type.forName(route.Package__c, route.Implementation__c);
            return implType.newInstance();
        }
        return null;
    }
}
```

## Package Versioning and Compatibility

### 1. Semantic Versioning Pattern

```json
{
  "package": "customer-management",
  "versionNumber": "2.1.3.NEXT",
  "dependencies": [
    {
      "package": "common",
      "versionNumber": "1.3.2.LATEST"  // Minimum version
    }
  ]
}
```

### 2. Backward Compatibility Pattern

```apex
// Version 1.0 interface
public interface ICustomerService {
    Customer getCustomer(Id customerId);
}

// Version 2.0 - Backward compatible
public interface ICustomerServiceV2 extends ICustomerService {
    Customer getCustomer(Id customerId);  // Original method
    Customer getCustomerWithHistory(Id customerId);  // New method
}

// Implementation supports both versions
public class CustomerServiceImpl implements ICustomerServiceV2 {
    public Customer getCustomer(Id customerId) {
        return getCustomerWithHistory(customerId).current;
    }
    
    public CustomerWithHistory getCustomerWithHistory(Id customerId) {
        // New implementation
    }
}
```

## Testing Package Boundaries

### 1. Contract Testing

```apex
@IsTest
public class CustomerServiceContractTest {
    @IsTest
    static void testServiceContract() {
        // Test that service implements expected interface
        ICustomerService service = (ICustomerService) 
            Application.Service.newInstance(ICustomerService.class);
        
        System.assertNotEquals(null, service, 'Service must be available');
        
        // Test contract methods exist and work
        Customer testCustomer = TestDataFactory.createCustomer();
        Customer result = service.getCustomer(testCustomer.Id);
        
        System.assertNotEquals(null, result, 'Service must return customer');
    }
}
```

### 2. Boundary Testing

```apex
@IsTest
public class PackageBoundaryTest {
    @IsTest
    static void testNoDirectDatabaseAccess() {
        // Verify package doesn't directly access other package's objects
        String packageName = 'customer-management';
        Set<String> allowedObjects = PackageRegistry.getAllowedObjects(packageName);
        
        for (String query : getPackageQueries(packageName)) {
            String objectName = extractObjectFromQuery(query);
            System.assert(
                allowedObjects.contains(objectName),
                'Package should not query ' + objectName
            );
        }
    }
}
```

## Package Boundary Checklist

### Design Phase
- [ ] Clear domain/capability definition
- [ ] Identified package dependencies
- [ ] Defined public interfaces
- [ ] Documented package responsibilities
- [ ] Established ownership

### Implementation Phase
- [ ] All public APIs through interfaces
- [ ] No direct cross-package database access
- [ ] Dependencies injected, not hardcoded
- [ ] Platform Events for async communication
- [ ] Package-specific configuration

### Testing Phase
- [ ] Contract tests for interfaces
- [ ] Mock implementations for dependencies
- [ ] Boundary violation tests
- [ ] Version compatibility tests
- [ ] Integration tests with dependencies

### Deployment Phase
- [ ] Dependency versions specified
- [ ] Deployment order documented
- [ ] Rollback plan prepared
- [ ] Package-specific permissions
- [ ] Post-deployment validation

## Common Boundary Violations

### 1. Database Coupling
```apex
// ❌ BAD: Direct query to another package's object
List<Order__c> orders = [SELECT Id FROM Order__c WHERE CustomerId__c = :custId];

// ✅ GOOD: Use service interface
List<Order> orders = orderService.getOrdersForCustomer(custId);
```

### 2. Shared Global Variables
```apex
// ❌ BAD: Global static variable
public static Map<Id, Customer> customerCache = new Map<Id, Customer>();

// ✅ GOOD: Package-private cache with service access
private static Map<Id, Customer> cache = new Map<Id, Customer>();
public Customer getCachedCustomer(Id customerId) {
    // Controlled access
}
```

### 3. Trigger Dependencies
```apex
// ❌ BAD: Trigger calls another package directly
trigger OrderTrigger on Order__c (after insert) {
    CustomerPackage.CustomerService.updateCustomerOrders(Trigger.new);
}

// ✅ GOOD: Trigger publishes event
trigger OrderTrigger on Order__c (after insert) {
    EventBus.publish(OrderEvents.created(Trigger.new));
}
```

## Conclusion

Clear package boundaries are essential for:
- **Maintainability**: Changes isolated to packages
- **Scalability**: Add packages without affecting others
- **Testability**: Test packages in isolation
- **Deployability**: Deploy packages independently
- **Team Autonomy**: Teams own clear boundaries

The key is finding the right balance between isolation and practicality for your specific context.