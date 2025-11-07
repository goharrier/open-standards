# Apex Anti-Patterns

Common Apex anti-patterns that should be avoided to ensure maintainable, efficient, and secure code.

## Do not keep unused code

If a piece of code is no longer referenced or used anywhere, it should be removed.
This includes code that has been commented out.
If a certain piece of code should be temporarily be disabled, consider using [feature flags](/technical/architecture-and-design-patterns/feature-flags.md) instead.

**Why this matters:**
- Reduces technical debt and confusion for future developers
- Improves org performance and reduces complexity
- Makes deployments cleaner and more predictable
- Reduces security surface area

### Do not hardcode IDs

Do not hardcode IDs (like Record Types, Users, Profiles, etc.)

**Why this matters:**
- IDs are environment-specific and will break during deployments
- Makes code fragile and difficult to maintain
- Prevents proper testing across different environments

**What not to do:**
```apex
// Bad - hardcoded Record Type ID
if (account.RecordTypeId == '0120000000001ABC') {
    // logic here
}

// Bad - hardcoded User ID
if (UserInfo.getUserId() == '0050000000001XYZ') {
    // logic here
}
```

**What to do instead:**
```apex
// Good - use Record Type developer name
Id recordTypeId = Schema.SObjectType.Account.getRecordTypeInfosByDeveloperName()
    .get('Business_Account').getRecordTypeId();

// Good - use Custom Permissions or Custom Settings
if (FeatureManagement.checkPermission('Special_Access_Permission')) {
    // logic here
}

// Good - use Custom Metadata Types for configuration
Configuration__mdt config = Configuration__mdt.getInstance('Default');
```

**Alternatives to hardcoding:**
- Use Record Type developer names with Schema methods
- Use Custom Permissions for user-based logic
- Use Custom Metadata Types for configuration data
- Use Custom Settings for environment-specific values

### Do not hardcode credentials

Do not hardcode credentials (e.g. passwords, API keys, etc.)

**Why this matters:**
- Exposes sensitive information in version control
- Creates serious security vulnerabilities
- Makes credential rotation difficult
- Violates security best practices and compliance requirements

**What to do instead:**

Use Named Credentials if possible. If not, store sensitive information in Custom Metadata Types or Custom Settings with restricted access.
