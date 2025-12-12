# Modular Package Architecture

## Intent

Enable large-scale Salesforce development through independent, composable packages that can be developed, tested, and deployed in isolation while maintaining clear boundaries and contracts between components.

## Core Principles

### 1. Package Layering Strategy

The architecture follows a strict layered approach where dependencies flow in one direction:

```
Environment-Specific → Access Management → Feature Packages → Domain Packages → Core → Common/Frameworks
```

### 2. Package Types and Responsibilities

#### Foundation Layer

* **common**: Shared frameworks (fflib, logger) with no business logic
* **core-unpackaged**: Base metadata that cannot be packaged

#### Core Layer

* **core**: Central business services, selectors, and domain classes
* Provides interfaces that other packages implement or consume
* Contains the main Application class for dependency injection

#### Domain Packages

* Feature-complete vertical slices (e.g., `revenue`, `lead-management`)
* Self-contained with their own Application\_XX class
* Depend only on core and common

#### Infrastructure Packages

* **env-specific-alias-pre**: Environment-specific configuration
* **access-management**: Security and permissions
* **ui**: User interface components

## Implementation Pattern

### Package Application Class

Each package defines its own Application class following this pattern:

```apex
// Application_XX.cls where XX is package abbreviation
public class Application_RE {
    // Package-specific Unit of Work
    public static final fflib_Application.UnitOfWorkFactory UnitOfWork =
        new fflib_Application.UnitOfWorkFactory(
            new List<SObjectType>{
                // Package-specific objects
            });

    // Service implementations mapping
    private static final Map<Type, Type> serviceImplementationTypeMapping = 
        new Map<Type, Type>{
            IContractLineItemsService_RE.class => ContractLineItemsServiceImpl_RE.class
        };

    // Service factory for this package
    public static final fflib_Application.ServiceFactory Service =
        new fflib_Application.ServiceFactory(serviceImplementationTypeMapping);

    // Selector factory for this package
    public static final fflib_Application.SelectorFactory Selector =
        new fflib_Application.SelectorFactory(
            new Map<SObjectType, Type>{
                ContractLineItem__c.SObjectType => ContractLineItemsSelector_RE.class
            });

    // Domain factory for this package  
    public static final fflib_Application.DomainFactory Domain =
        new fflib_Application.DomainFactory(
            Application_RE.Selector,
            new Map<SObjectType, Type>{
                ContractLineItem__c.SObjectType => ContractLineItems_RE.Constructor.class
            });
}
```

### Service Interface Pattern

Services expose functionality through interfaces:

```apex
// In core package
public interface ICasesService {
    void setEndTimeOnOpenStatusChanges(ICases cases);
    void insertNewStatusChange(ICases cases);
}

// Implementation in core package
public class CasesServiceImpl implements ICasesService {
    // Implementation details
}

// Consuming from another package
ICasesService casesService = (ICasesService) Application.Service.newInstance(ICasesService.class);
```

### Cross-Package Communication

Packages communicate through:

1. **Service Interfaces** - Synchronous calls through defined contracts
2. **Platform Events** - Asynchronous, loosely coupled communication
3. **Custom Metadata** - Configuration-driven behavior

## Package Configuration

### sfdx-project.json Structure

```json
{
  "packageDirectories": [
    {
      "path": "src-env-specific-alias-pre",
      "package": "env-specific-alias-pre",
      "aliasfy": true,  // Environment-specific aliasing
      "versionNumber": "1.1.0.NEXT"
    },
    {
      "path": "./src/core",
      "package": "core",
      "seedMetadata": {
        "path": "./src/core-unpackaged"  // Unpackaged dependencies
      },
      "enableFHT": true,  // Field History Tracking
      "dependencies": [
        {
          "package": "common",
          "versionNumber": "1.3.2.LATEST"
        }
      ]
    },
    {
      "path": "./src/service",
      "package": "service",
      "dependencies": [
        {
          "package": "common",
          "versionNumber": "1.3.2.LATEST"
        },
        {
          "package": "core",
          "versionNumber": "1.8.27.LATEST"
        }
      ]
    }
  ]
}
```

### Environment-Specific Configuration

```
src-env-specific-alias-pre/
├── default/           # Default configuration
├── dev/              # Development-specific
├── staging/          # Staging-specific
└── prod/             # Production-specific
    ├── namedCredentials/
    ├── customMetadata/
    └── settings/
```

## Deployment Strategy

### Release Configuration

```yaml
releaseName: all-packages
includeOnlyArtifacts:
  - env-specific-alias-pre    # Environment config first
  - common                     # Frameworks
  - core                       # Core business logic
  - service                    # Domain services
  - ui                        # UI components
  - access-management         # Permissions last
releasedefinitionProperties:
  promotePackagesBeforeDeploymentToOrg: prod
  skipIfAlreadyInstalled: true
```

### Package Deployment Order

1. Environment-specific configuration (`aliasfy: true`)
2. Foundation packages (common, frameworks)
3. Core business packages
4. Feature packages
5. UI packages (`alwaysDeploy: true`)
6. Access management packages

## Benefits

### Development Benefits

* **Parallel Development**: Teams work on packages independently
* **Clear Ownership**: Each package has defined boundaries
* **Faster CI/CD**: Deploy only changed packages
* **Isolated Testing**: Test packages in isolation with mocks

### Architectural Benefits

* **Loose Coupling**: Packages communicate through interfaces
* **High Cohesion**: Related functionality grouped together
* **Reusability**: Packages can be shared across orgs
* **Maintainability**: Changes isolated to specific packages

## Anti-Patterns to Avoid

### 1. Circular Dependencies

```apex
// ❌ BAD: Package A depends on B, B depends on A
// Package A
public class ServiceA {
    ServiceB b = new ServiceB();  // Direct dependency
}

// Package B  
public class ServiceB {
    ServiceA a = new ServiceA();  // Circular!
}

// ✅ GOOD: Use interfaces and events
// Package A implements interface from Common
public class ServiceA implements IServiceA {
    // Publish event instead of direct call
    EventBus.publish(new ServiceAEvent__e());
}
```

### 2. Package Sprawl

```
// ❌ BAD: Too many small packages
src/
├── validate-email-package/     # 2 classes
├── format-phone-package/       # 1 class  
├── calculate-tax-package/      # 3 classes

// ✅ GOOD: Cohesive domain packages
src/
├── contact-management/          # All contact-related functionality
    ├── validation/
    ├── formatting/
    └── services/
```

### 3. Hidden Coupling

```apex
// ❌ BAD: Packages coupled through shared state
// Package A writes to Custom Setting
CustomSetting__c.getInstance().Value__c = 'data';

// Package B reads from same Custom Setting
String value = CustomSetting__c.getInstance().Value__c;

// ✅ GOOD: Explicit service contract
public interface IConfigurationService {
    String getValue(String key);
    void setValue(String key, String value);
}
```

## Package Sizing Guidelines

### When to Create a New Package

* Distinct business domain (10+ related objects)
* Different deployment cadence
* Separate team ownership
* Reusable across multiple orgs

### When to Keep in Existing Package

* Tightly coupled functionality
* Shared transaction boundaries
* < 5 objects or 20 classes
* Same deployment lifecycle

## Testing Strategy

### Unit Testing

```apex
@IsTest
private class ServiceTest {
    @IsTest
    static void testWithMockedDependency() {
        // Mock external package dependency
        fflib_ApexMocks mocks = new fflib_ApexMocks();
        IExternalService mockService = mocks.mock(IExternalService.class);
        
        // Inject mock
        Application.Service.setMock(IExternalService.class, mockService);
        
        // Test in isolation
        // ...
    }
}
```

### Integration Testing

* Deploy packages to scratch org in correct order
* Run integration test suite across packages
* Validate cross-package platform events

## Migration Path

### From Monolithic to Modular

1. **Identify Boundaries**: Map existing code to domains
2. **Extract Interfaces**: Define service contracts
3. **Create Package Structure**: Set up package directories
4. **Move Code Incrementally**: One domain at a time
5. **Update Dependencies**: Adjust sfdx-project.json
6. **Test Thoroughly**: Ensure no regression

## Production Implementation Patterns

Successful implementations of this architecture demonstrate:

* **Dozens of packages** managed independently in production systems
* **Clear separation** between foundation, core, domain, and UI layers
* **Environment-specific** configuration packages using `aliasfy: true` for environment-based deployment
* **Package-specific Application classes** following the Application\_XX pattern for each domain
* **Successful scaling** to large development teams working in parallel without conflicts
