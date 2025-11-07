# General Anti-Patterns

Common general anti-patterns that should be avoided to ensure maintainable, efficient, and secure Salesforce implementations.

## Do not keep unused metadata

Do not keep unused metadata (fields, Flows, classes, etc.) unless there's a very specific reason.

**Why this matters:**
- Reduces technical debt and confusion for future developers
- Improves org performance and reduces complexity
- Makes deployments cleaner and more predictable
- Reduces security surface area

**Examples of unused metadata to remove:**
- Unused custom fields on objects
- Inactive or obsolete Flows
- Deprecated Apex classes and triggers
- Unused custom objects
- Outdated validation rules
- Unused Lightning components

**Exceptions:**
- Metadata required for data migration processes
- Fields/objects needed for audit trails or compliance
- Components that are temporarily disabled but will be reactivated

## Avoid using record-triggered flows

Avoid using record-triggered flows (especially after-save record triggered flows, which have significant overhead). Instead, use a trigger framework.

**Why this matters:**
- Record-triggered flows have substantial performance overhead compared to Apex triggers
- Flows are harder to debug and maintain for complex business logic
- Trigger frameworks provide better control over execution order and bulkification
- Better testability and code reusability with Apex-based solutions

**Reference:**
- [Salesforce Architect Decision Guide: Trigger Automation](https://architect.salesforce.com/decision-guides/trigger-automation)

**Recommended trigger frameworks:**
- [fflib Apex Extensions](https://github.com/wimvelzeboer/fflib-apex-extensions)
- [Trigger Actions Framework](https://github.com/mitchspano/trigger-actions-framework)

**Migration strategy:**
- Identify existing record-triggered flows in your org
- Prioritize migration based on performance impact and complexity
- Implement trigger framework gradually
- Test thoroughly before deactivating flows

## Avoid using personal users for automated processes

When scheduling automated jobs, generally use a generic user account as the executor rather than personal user accounts.

**Why:**
- **Operational Continuity**: If done with a person-tied user, if their user is deactivated, the process will stop working.
- **Clear Audit Trail**: It's easier to track changes (like record updates) made by automations vs real humans.

**Examples:**
- Bot accounts: "Integration Bot", "Report Bot"
- Service accounts: "Data Sync Service", "Backup Service"
