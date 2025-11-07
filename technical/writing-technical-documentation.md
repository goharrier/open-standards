# Writing and Maintaining Technical Documentation

Effective technical documentation is a critical component of maintainable software systems. Well-written documentation reduces onboarding time, minimizes support burden, and serves as a single source of truth for architectural decisions and implementation patterns.

## Documentation Philosophy

### Write for Your Audience

Technical documentation serves multiple audiences with different needs:

- New developers: Need conceptual overviews, getting started guides, and common patterns
- Experienced team members: Need detailed API references, edge cases, and optimization techniques
- Operations teams: Need runbooks, troubleshooting guides, and monitoring procedures
- Architects: Need system diagrams, integration patterns, and design decisions

Tailor content depth and terminology to your target audience. Avoid assuming knowledge that new team members won't have, but don't over-explain concepts that experienced developers already understand.

### Documentation as Code

Treat documentation with the same rigor as code:
- **Version control**: Store documentation in the same repository as code.
- **Review process**: Require peer review for documentation changes.
- **Broken link checking**: Validate internal and external references.
- **Search optimization**: Structure content for discoverability.

### Living Documentation

Documentation decays without maintenance:

- **Update on code changes**: Documentation updates are part of definition of done.
- **Regular reviews**: Schedule periodic audits to ensure accuracy.
- **Deprecation markers**: Clearly flag outdated content with removal dates.

## Style Guide Foundations

Follow [Google's Technical Writing Style Guide](https://developers.google.com/style) as your baseline, with the following Salesforce-specific adaptations.

### Terminology Standards

Maintain consistent terminology:

#### Salesforce-specific terms:

- Use "org" or "organization", not "instance" or "environment".
- Use "record" not "row" or "entry".
- Use "field" not "column" or "attribute".

#### Code element formatting:

- Class names: `AccountTriggerHandler`
- Method names: `processRecords()`
- Field names: `Account.AnnualRevenue`
- sObject API names: `Account`, `Opportunity`, `CustomObject__c`
