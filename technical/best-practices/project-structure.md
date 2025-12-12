# Salesforce Project Structure

{% hint style="info" %}
### TODO

Merge this content with [Class Folder Organization Patterns](class-folder-organization.md), and complete the content here.
{% endhint %}

### Module-Based Architecture

In Salesforce development, the lack of traditional package systems and folder structures presents unique challenges. All Apex classes exist in a global namespace, making code organization and modularity crucial for maintainable enterprise applications.

Our Salesforce projects follow a module-based architecture where functionality is organized into discrete modules within the `src` folder. Each module represents a specific business domain or technical capability.

Example project structure:

```
src/
├── core/
│   ├── main/default/classes/
│   │   ├── Application.cls
│   │   ├── services/
│   │   │   └── CasesService.cls
│   │   └── triggerActions/
│   │       └── TA_Case_InitDefaults.cls
│   └── test/default/classes/
│       ├── services/
│       │   └── CasesServiceTest.cls
│       └── triggerActions/
│           └── TA_Case_InitDefaultsTest.cls
├── document-explorer/
│   ├── main/default/classes/
│   │   ├── Application_DE.cls
│   │   ├── services/
│   │   │   └── DocumentService_DE.cls
│   └── test/default/classes/
│       ├── services/
│       │   └── DocumentServiceTest_DE.cls
└── lead-management/
    ├── main/default/classes/
    │   ├── Application_LM.cls
    │   ├── services/
    │   │   └── LeadService_LM.cls
    └── test/default/classes/
        ├── services/
        │   └── LeadServiceTest_PM.cls
```

Since Salesforce operates with a global class namespace, we use consistent naming patterns and suffixes to maintain module isolation and code clarity.

**Exception:** Classes in the `core` module do not use suffixes.

**Benefits of This Approach:**

1. Module Isolation: Each module contains related functionality, making it easier to understand and maintain specific business domains.
2. Reduced Naming Conflicts: Descriptive suffixes minimize the risk of class name collisions across different modules.
3. Enhanced Testability: Module boundaries make it easier to write focused unit tests and mock dependencies.

### Standard Class Organization Patterns

To further classify and organize classes within each module, we use standardized folder structures and naming patterns. These are some of the most common folders, but modules are not limited to just these organizational patterns:

```
module-name/
├── main/default/classes/
│   ├── constants/
│   ├── domains/
│   │   ├── interfaces/
│   │   │   └── IAccounts.cls
│   │   └── Accounts.cls
│   ├── selectors/
│   │   ├── interfaces/
│   │   │   └── IAccountsSelector.cls
│   │   └── AccountsSelector.cls
│   ├── services/
│   │   ├── implementations/
│   │   │   └── DocumentsServiceImpl.cls
│   │   ├── interfaces/
│   │   │   └── IDocumentsService.cls
│   │   └── DocumentsService.cls
│   ├── triggerActions/
│   └── Application.cls
```

**Folder Descriptions:**

* `constants/`: Static values and configuration constants
* `domains/`: Business logic and domain models with their interfaces
* `selectors/`: Data access layer classes with their interfaces
* `services/`: Business service layer with implementations and interfaces
* `triggerActions/`: Trigger handler classes

> **Note:** For a deeper understanding of the domains, selectors, services, and trigger actions architectural patterns, refer to the [fflib - Apex Framework](../frameworks/fflib-apex-framework.md) documentation.
