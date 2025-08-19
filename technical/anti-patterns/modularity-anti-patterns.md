# Modularity Anti-Patterns in Salesforce

## Overview

This document identifies common anti-patterns in modular Salesforce architectures based on real-world refactoring experiences. Each anti-pattern includes the problem, why it occurs, its impact, and proven solutions.

## 1. The Monolithic Package Anti-Pattern

### The Intent Behind Modularity

The whole point of package-based architecture is to enable **independent development, testing, and deployment** of features. When everything lives in one package, you lose these benefits entirely. Think of it like building a house where every room's electrical system is connected - you can't work on the kitchen wiring without shutting down the entire house.

### Problem

A single package grows to contain hundreds of components, mixing multiple domains and responsibilities. What starts as a "core" package becomes a dumping ground for everything.

### How It Manifests

```
src/
└── mega-package/
    ├── classes/           # 500+ classes: OrderService, CustomerService, 
    │                     # InventoryManager, TaxCalculator, EmailHandler...
    ├── objects/          # 50+ objects from unrelated domains
    ├── flows/            # Flows for customer onboarding, order processing,
    │                     # inventory management, reporting...
    └── lwc/              # UI components for every feature in the system
```

### Why This Is Problematic

Imagine you need to fix a bug in the tax calculation logic. In a monolithic package:
1. You deploy the entire package (30+ minutes)
2. All tests run (45+ minutes) 
3. Any team working on ANY feature is blocked
4. If something breaks, you rollback EVERYTHING
5. You can't give a client just the tax fix - they get all pending changes

### The Real Cost

```apex
// What looks like a simple change...
public class TaxCalculator {
    public Decimal calculateTax(Decimal amount) {
        // return amount * 0.08;  // OLD
        return amount * 0.085;     // NEW: Updated tax rate
    }
}

// ...requires deploying all of this:
// - 500 other classes
// - Unfinished features in development
// - Experimental code
// - Changes from 5 other teams
// Risk level: EXTREME
```

### Solution: Domain-Driven Package Decomposition

The solution is to break apart the monolith based on **business capabilities**, not technical layers. Each package should represent something a business person would understand.

```
src/
├── tax-management/           # JUST tax-related functionality
│   ├── classes/             
│   │   ├── TaxCalculator.cls
│   │   ├── TaxRuleEngine.cls
│   │   └── TaxReportService.cls
│   └── objects/
│       └── TaxRate__c
│
├── order-management/         # JUST order-related functionality  
│   ├── classes/
│   │   ├── OrderService.cls
│   │   ├── OrderValidator.cls
│   │   └── OrderFulfillment.cls
│   └── objects/
│       ├── Order__c
│       └── OrderItem__c
│
└── customer-management/      # JUST customer-related functionality
    ├── classes/
    │   ├── CustomerService.cls
    │   └── CustomerSegmentation.cls
    └── objects/
        └── CustomerProfile__c
```

Now that tax rate change:
- Deploys in 2 minutes (just tax-management package)
- Only runs tax-related tests
- Other teams continue working
- Can be deployed to specific orgs
- Rollback affects only tax functionality

## 2. The False Modularity Anti-Pattern

### The Intent We're Violating

True modularity means packages can be **understood, developed, tested, and deployed in isolation**. When packages secretly depend on each other through the database or configuration, they're lying about being modular.

### Problem

Packages appear modular but are tightly coupled through hidden dependencies. It's like having "separate" apartments that share plumbing - turn off water in one, and the neighbor's shower stops working.

### How It Manifests

```apex
// pricing-package - "Independent" package
public class PricingService {
    public Decimal calculatePrice(Id productId) {
        // Hidden dependency #1: Directly queries another package's object
        Product2 product = [SELECT Cost__c, Markup__c, Category__c 
                          FROM Product2 WHERE Id = :productId];
        
        // Hidden dependency #2: Assumes field exists and has specific values
        if (product.Category__c == 'Premium') {  // What if inventory package changes this?
            
            // Hidden dependency #3: Reads another package's custom settings
            Decimal globalDiscount = InventorySettings__c.getInstance().GlobalDiscount__c;
            
            // Hidden dependency #4: Queries another package's custom metadata
            PricingRule__mdt rule = [SELECT Multiplier__c FROM PricingRule__mdt 
                                    WHERE Category__c = :product.Category__c];
            
            return product.Cost__c * product.Markup__c * rule.Multiplier__c - globalDiscount;
        }
    }
}

// inventory-package - Also "independent"
public class InventoryService {
    public void updateProductCategory() {
        // Changes the field that pricing depends on!
        Product2 product = [SELECT Category__c FROM Product2 WHERE Id = :productId];
        product.Category__c = 'Standard';  // Breaks pricing calculation
        update product;
        
        // Updates settings that pricing reads
        InventorySettings__c.getInstance().GlobalDiscount__c = null;
    }
}
```

### Why This Is Dangerous

The code above creates an **illusion of modularity**. The packages seem independent but:
- You can't test PricingService without inventory package's data
- Deploying inventory changes can break pricing without warning
- There's no contract defining what Category__c values are valid
- The dependency is invisible in package manifests

### The Hidden Coupling Problem

```apex
// What the deployment manifest shows:
// pricing-package:
//   dependencies: []  ← Looks independent!
// 
// inventory-package:
//   dependencies: []  ← Also looks independent!

// What actually happens in production:
// Day 1: Deploy pricing-package → Works
// Day 2: Deploy inventory-package → Works
// Day 3: Inventory team changes Category__c picklist values
// Day 4: All pricing calculations fail silently
// Day 5: Nobody knows why revenue reports are wrong
```

### Solution: Explicit Contracts and Ownership

Make dependencies visible and controlled through interfaces:

```apex
// shared-interfaces package - The CONTRACT
public interface IProductDataProvider {
    ProductInfo getProductInfo(Id productId);
}

public class ProductInfo {
    public Decimal cost {get; set;}
    public Decimal markup {get; set;}
    public String pricingCategory {get; set;}  // Not the raw field!
}

// pricing-package - Explicit dependency
public class PricingService {
    // Dependency injection makes it visible and testable
    @TestVisible
    private IProductDataProvider productProvider {
        get {
            if (productProvider == null) {
                productProvider = (IProductDataProvider) 
                    Application.Service.newInstance(IProductDataProvider.class);
            }
            return productProvider;
        }
        set;
    }
    
    public Decimal calculatePrice(Id productId) {
        // Use the contract, not direct queries
        ProductInfo info = productProvider.getProductInfo(productId);
        
        // Now we're working with a stable interface
        if (info.pricingCategory == 'PREMIUM') {
            return info.cost * info.markup * getPremiumMultiplier();
        }
    }
}

// inventory-package - Implements the contract
public class ProductDataService implements IProductDataProvider {
    public ProductInfo getProductInfo(Id productId) {
        Product2 product = [SELECT Cost__c, Markup__c, Category__c FROM Product2 WHERE Id = :productId];
        
        // Translates internal data to contract
        return new ProductInfo()
            .setCost(product.Cost__c)
            .setMarkup(product.Markup__c)
            .setPricingCategory(mapCategoryToPricingCategory(product.Category__c));
    }
    
    // Internal changes don't break contract
    private String mapCategoryToPricingCategory(String category) {
        // Can change internal categories without breaking pricing
        Map<String, String> categoryMap = new Map<String, String>{
            'Premium' => 'PREMIUM',
            'Deluxe' => 'PREMIUM',    // New category, same pricing
            'Standard' => 'STANDARD'
        };
        return categoryMap.get(category);
    }
}
```

## 3. The Circular Dependency Maze

### The Intent of Dependency Management

Dependencies should flow in **one direction**, typically from higher-level packages (business logic) to lower-level packages (utilities, data access). When packages depend on each other circularly, it's like two people holding doors open for each other - nobody can actually go through.

### The Problem Visualized

```
   sales ──depends on──> inventory
     ↑                      ↓
     │                      │
   depends                depends
     on                     on
     │                      │
     └──── pricing <────────┘
     
Result: NOBODY CAN DEPLOY!
```

### Real Code That Creates Circles

```apex
// sales package - Needs inventory data
public class QuoteService {
    public Decimal calculateQuoteTotal(Id quoteId) {
        Quote__c quote = [SELECT Id, (SELECT Product__c, Quantity__c FROM QuoteItems__r) FROM Quote__c];
        
        for (QuoteItem__c item : quote.QuoteItems__r) {
            // DEPENDENCY: Sales → Inventory
            Boolean available = InventoryService.checkAvailability(item.Product__c, item.Quantity__c);
            if (!available) {
                throw new QuoteException('Product not available');
            }
        }
        return total;
    }
}

// inventory package - Needs pricing data
public class InventoryService {
    public static Boolean checkAvailability(Id productId, Decimal quantity) {
        // DEPENDENCY: Inventory → Pricing
        Decimal currentPrice = PricingService.getCurrentPrice(productId);
        if (currentPrice > 1000) {
            // High-value items need special handling
            return checkPremiumInventory(productId, quantity);
        }
        return checkStandardInventory(productId, quantity);
    }
}

// pricing package - Needs sales data
public class PricingService {
    public static Decimal getCurrentPrice(Id productId) {
        // DEPENDENCY: Pricing → Sales (CIRCULAR!)
        Decimal basePrice = getBasePrice(productId);
        Decimal discountRate = QuoteService.getVolumeDiscount(productId);
        return basePrice * (1 - discountRate);
    }
}
```

### Why Circular Dependencies Are Deadly

1. **Can't deploy**: Package A needs B, B needs C, C needs A... infinite loop
2. **Can't test**: Mocking becomes impossible when everything depends on everything
3. **Can't understand**: Where does the logic actually live?
4. **Can't refactor**: Moving anything breaks everything

### Solution: Dependency Inversion and Events

Break the cycle by introducing abstractions and using events for loose coupling:

```apex
// core-interfaces package - No dependencies, just contracts
public interface IPricingProvider {
    Decimal getCurrentPrice(Id productId);
    Decimal getBasePrice(Id productId);
}

public interface IInventoryProvider {
    Boolean checkAvailability(Id productId, Decimal quantity);
    InventoryStatus getStatus(Id productId);
}

public interface IDiscountProvider {
    Decimal getVolumeDiscount(Id productId, Decimal quantity);
}

// pricing package - Depends only on interfaces
public class PricingService implements IPricingProvider {
    // Inject discount provider instead of calling sales directly
    @TestVisible
    private IDiscountProvider discountProvider;
    
    public Decimal getCurrentPrice(Id productId) {
        Decimal basePrice = getBasePrice(productId);
        // No more circular dependency!
        Decimal discount = discountProvider.getVolumeDiscount(productId, currentQuantity);
        return basePrice * (1 - discount);
    }
}

// sales package - Implements discount logic
public class SalesDiscountService implements IDiscountProvider {
    public Decimal getVolumeDiscount(Id productId, Decimal quantity) {
        // Sales-specific discount logic
        if (quantity > 100) return 0.15;
        if (quantity > 50) return 0.10;
        return 0.05;
    }
}

// inventory package - Uses events for loose coupling
public class InventoryService implements IInventoryProvider {
    public Boolean checkAvailability(Id productId, Decimal quantity) {
        // Instead of calling pricing directly, publish event
        InventoryCheckRequest__e request = new InventoryCheckRequest__e(
            ProductId__c = productId,
            Quantity__c = quantity,
            RequestId__c = generateRequestId()
        );
        
        EventBus.publish(request);
        
        // Wait for response or use async pattern
        return getInventoryResponse(request.RequestId__c);
    }
}
```

## 4. The Chatty Packages Anti-Pattern

### The Intent of Service Boundaries

Package interfaces should be **coarse-grained** - think of them like international phone calls. You wouldn't call another country to ask one word at a time; you'd have a complete conversation. Same with packages.

### Problem Illustrated

```apex
// order-processing package - The Chatty Customer
public class OrderProcessor {
    public void processLargeOrder(Order__c order) {
        // Processing 100 items means 500+ cross-package calls!
        for (OrderItem__c item : order.OrderItems__r) {
            // Call 1: Check price (→ pricing package)
            Decimal price = PricingService.getPrice(item.Product__c);
            
            // Call 2: Check inventory (→ inventory package)  
            Boolean available = InventoryService.checkOne(item.Product__c);
            
            // Call 3: Get tax (→ tax package)
            Decimal tax = TaxService.calculateForProduct(item.Product__c);
            
            // Call 4: Check shipping (→ logistics package)
            String method = ShippingService.getMethodForProduct(item.Product__c);
            
            // Call 5: Validate address (→ logistics package again!)
            Boolean validAddress = ShippingService.validateAddress(order.ShipTo__c);
            
            // 5 calls × 100 items = 500 cross-package calls
            // This is architectural diabetes!
        }
    }
}
```

### Why Chattiness Kills Performance

Each cross-package call has overhead:
- Parameter marshalling
- Service location/injection
- Security checks
- Logging/monitoring
- Error handling

Multiply that by 500 and you have:
- Slow performance
- Governor limit issues
- Difficult debugging
- Complex test setup

### Solution: Bulk Operations and Aggregated Interfaces

Design interfaces that accept collections and return complete results:

```apex
// Single request object with everything needed
public class OrderValidationRequest {
    public Order__c order {get; set;}
    public List<OrderItem__c> items {get; set;}
    public Id accountId {get; set;}
    public Address shippingAddress {get; set;}
}

// Single response with all results
public class OrderValidationResponse {
    public Map<Id, PricingInfo> pricing {get; set;}
    public Map<Id, InventoryInfo> inventory {get; set;}
    public Map<Id, TaxInfo> taxes {get; set;}
    public ShippingValidation shipping {get; set;}
    
    public class PricingInfo {
        public Decimal basePrice {get; set;}
        public Decimal discount {get; set;}
        public Decimal finalPrice {get; set;}
    }
    
    public class InventoryInfo {
        public Boolean available {get; set;}
        public Decimal quantity {get; set;}
        public Date expectedDate {get; set;}
    }
}

// Coarse-grained interface - ONE call instead of 500
public interface IOrderValidationService {
    OrderValidationResponse validateOrder(OrderValidationRequest request);
}

// Implementation handles all coordination internally
public class OrderValidationServiceImpl implements IOrderValidationService {
    public OrderValidationResponse validateOrder(OrderValidationRequest request) {
        // Extract all product IDs once
        Set<Id> productIds = extractProductIds(request.items);
        
        // Make ONE bulk call to each service
        Map<Id, PricingInfo> pricing = PricingService.getPricingForProducts(productIds);
        Map<Id, InventoryInfo> inventory = InventoryService.checkBulkAvailability(productIds);
        Map<Id, TaxInfo> taxes = TaxService.calculateBulkTaxes(productIds, request.shippingAddress);
        ShippingValidation shipping = ShippingService.validateShipping(request.shippingAddress, productIds);
        
        // Return complete response
        return new OrderValidationResponse()
            .withPricing(pricing)
            .withInventory(inventory)
            .withTaxes(taxes)
            .withShipping(shipping);
    }
}

// Now the order processor is clean
public class OrderProcessor {
    public void processLargeOrder(Order__c order) {
        // ONE call for everything
        OrderValidationRequest request = buildRequest(order);
        OrderValidationResponse response = validationService.validateOrder(request);
        
        // Process with complete information
        for (OrderItem__c item : order.OrderItems__r) {
            PricingInfo pricing = response.pricing.get(item.Product__c);
            InventoryInfo inventory = response.inventory.get(item.Product__c);
            // Use the aggregated data
        }
    }
}
```

## 5. The Configuration Coupling Anti-Pattern

### The Intent of Package Independence

Each package should **own its configuration** and not be affected by other packages' configuration changes. Shared configuration is like sharing a toothbrush - it seems convenient until someone changes how they use it.

### The Hidden Configuration Problem

```apex
// notification package
public class EmailService {
    public void sendOrderConfirmation(Id orderId) {
        // Reads "system-wide" settings
        SystemSettings__c settings = SystemSettings__c.getOrgDefaults();
        
        String fromAddress = settings.EmailFromAddress__c;  // Who owns this?
        Integer retryCount = settings.MaxRetries__c;        // Is this for email?
        Boolean debugMode = settings.DebugEnabled__c;       // Or global debug?
        
        // What happens when another package changes these?
    }
}

// integration package - Different team, different purpose
public class APIService {
    public void configureSystem() {
        SystemSettings__c settings = SystemSettings__c.getOrgDefaults();
        
        // Changes settings for API, breaks email!
        settings.MaxRetries__c = 10;      // Email now retries 10 times
        settings.DebugEnabled__c = true;  // Email starts logging everything
        settings.EmailFromAddress__c = 'api@company.com';  // Wrong sender!
        
        update settings;
    }
}
```

### Why Shared Configuration Breaks Modularity

1. **No ownership**: Who's responsible for EmailFromAddress__c?
2. **Hidden dependencies**: Package manifest doesn't show config dependencies
3. **Runtime surprises**: Config changes break unrelated packages
4. **Testing nightmare**: Tests need specific config states

### Solution: Package-Specific Configuration

Each package owns its configuration with clear namespacing:

```apex
// Custom metadata for email package
public class EmailConfig__mdt {
    public String FromAddress__c {get; set;}
    public Integer MaxRetries__c {get; set;}
    public Boolean DebugMode__c {get; set;}
}

// Custom metadata for API package  
public class APIConfig__mdt {
    public String Endpoint__c {get; set;}
    public Integer Timeout__c {get; set;}
    public Integer MaxRetries__c {get; set;}  // Different from email retries!
}

// Email package configuration service
public class EmailConfigService {
    private static final String CONFIG_NAME = 'Default';
    
    public static EmailConfig__mdt getConfig() {
        EmailConfig__mdt config = EmailConfig__mdt.getInstance(CONFIG_NAME);
        
        // Package-specific defaults
        if (config == null) {
            config = new EmailConfig__mdt(
                FromAddress__c = 'noreply@company.com',
                MaxRetries__c = 3,
                DebugMode__c = false
            );
        }
        return config;
    }
}

// API package configuration service
public class APIConfigService {
    private static final String CONFIG_NAME = 'Default';
    
    public static APIConfig__mdt getConfig() {
        APIConfig__mdt config = APIConfig__mdt.getInstance(CONFIG_NAME);
        
        // Completely independent configuration
        if (config == null) {
            config = new APIConfig__mdt(
                Endpoint__c = 'https://api.company.com',
                Timeout__c = 30000,
                MaxRetries__c = 5  // Different retry logic for APIs
            );
        }
        return config;
    }
}
```

## 6. The Overly Generic Package Anti-Pattern

### The Intent of Domain Focus

Packages should solve **specific problems well**, not all problems poorly. It's better to have a sharp knife and a good screwdriver than a dull Swiss Army knife.

### The Generic Monster

```apex
// The "do everything" processor that does nothing well
public class UniversalProcessor {
    public Object process(Map<String, Object> request) {
        String operation = (String) request.get('operation');
        String objectType = (String) request.get('objectType');
        Map<String, Object> params = (Map<String, Object>) request.get('params');
        
        // The IF-ELSE pyramid of doom
        if (objectType == 'Order') {
            if (operation == 'Create') {
                if (params.containsKey('fastTrack')) {
                    if ((Boolean) params.get('fastTrack')) {
                        // 50 lines of fast track order creation
                    } else {
                        // 100 lines of normal order creation
                    }
                } else if (params.containsKey('bulk')) {
                    // 200 lines of bulk order processing
                }
            } else if (operation == 'Update') {
                // 300 lines of update logic
            } else if (operation == 'Cancel') {
                // 150 lines of cancellation
            }
        } else if (objectType == 'Invoice') {
            if (operation == 'Generate') {
                // 400 lines of invoice generation
            } else if (operation == 'Send') {
                // 200 lines of sending logic
            }
        } else if (objectType == 'Report') {
            // Another 1000 lines...
        }
        // Total: 3000+ lines of tangled logic
    }
}
```

### Why Generic Packages Fail

1. **Impossible to test**: Every change requires testing all paths
2. **Impossible to understand**: What does this package actually do?
3. **Impossible to maintain**: Where do you add new functionality?
4. **Terrible performance**: Constant branching and type checking
5. **No clear API**: What parameters are valid? Who knows!

### Solution: Domain-Specific Packages

Create focused packages with clear purposes:

```apex
// order-management package - Clear purpose
public interface IOrderService {
    Order__c createOrder(OrderCreationRequest request);
    Order__c createFastTrackOrder(FastTrackOrderRequest request);
    List<Order__c> createBulkOrders(List<OrderCreationRequest> requests);
    void updateOrder(Id orderId, OrderUpdateRequest updates);
    void cancelOrder(Id orderId, CancellationReason reason);
}

// invoice-generation package - Separate concern
public interface IInvoiceService {
    Invoice__c generateInvoice(InvoiceGenerationRequest request);
    void sendInvoice(Id invoiceId, InvoiceDeliveryOptions options);
    Blob renderInvoiceAsPDF(Id invoiceId);
}

// reporting package - Another clear domain
public interface IReportingService {
    Report generateSalesReport(DateRange period);
    Report generateInventoryReport(Set<Id> warehouseIds);
    void scheduleReport(ReportScheduleRequest schedule);
}

// Each service has:
// - Clear purpose (order management, invoicing, reporting)
// - Specific methods (not generic "process")
// - Typed parameters (not Map<String, Object>)
// - Testable interface (mock one service, not the universe)
// - Maintainable size (300 lines, not 3000)
```

## Key Takeaways

### The Business Test
Can you explain what a package does to a non-technical stakeholder? 
- ✅ "This handles tax calculations"
- ❌ "This processes various operations on multiple object types"

### The Deployment Test
Can you deploy a package independently without breaking others?
- ✅ Deploy tax changes without touching orders
- ❌ Deploy mega-package and pray nothing breaks

### The Team Test
Can two teams work on different packages without conflicts?
- ✅ Team A on orders, Team B on inventory, no merge conflicts
- ❌ Everyone fighting over the same files

### The Understanding Test
Can a new developer understand a package's purpose in 5 minutes?
- ✅ "order-management does... order management"
- ❌ "universal-processor does... everything?"

## Conclusion

These anti-patterns emerge naturally as systems grow. The key is to:
1. **Recognize them early** through metrics and code reviews
2. **Refactor incrementally** - you don't have to fix everything at once
3. **Prevent recurrence** through architecture governance
4. **Balance modularity with practicality** - some coupling is acceptable

Remember: The goal isn't perfect modularity - it's **maintainable, understandable, and deployable** systems that teams can work on without stepping on each other's toes.