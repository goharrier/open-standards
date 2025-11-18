# Nebula Logger - Logging

## Links

**Codebase**\
[https://github.com/jongpie/NebulaLogger](https://github.com/jongpie/NebulaLogger)

## Use case

Nebula Logger addresses the critical need for comprehensive logging capabilities within the Salesforce platform that go beyond the standard debug logs. It serves as a robust solution for tracking, monitoring, and analyzing activities and errors across your Salesforce org. The framework allows development teams to:

* Capture detailed information about errors, transactions, and system processes
* Maintain a persistent record of system activities that won't be auto-purged
* Relate log entries to specific Salesforce records for improved troubleshooting
* Control logging verbosity at the org, profile, or user level
* Support logging across all automation types: Apex, Flow, and Lightning Components (LWC & Aura)
* Track related logs across asynchronous processes and transactions
* Automatically capture contextual information about the org, user, and record
* Monitor platform events in real-time through the "Log Entry Event Stream" tab
* Apply data masking rules to protect sensitive information like SSNs and credit card numbers
* Organize and categorize logs using scenarios and tagging systems
* Extend functionality through the plugin framework for integration with external systems

This framework fills significant gaps in Salesforce's native logging capabilities, enabling teams to proactively monitor system health, identify issues before they impact users, and provide detailed information for troubleshooting when problems occur.

{% hint style="info" %}
To see how to use Nebula Logger in practice, refer to the [Logging best practices](/technical/best-practices/logging.md) documentation.
{% endhint %}

## Rationale

Our team has chosen Nebula Logger over other logging frameworks for several reasons:

1. **Native Salesforce Integration**: Unlike external logging solutions, Nebula Logger stores data within the Salesforce platform, allowing us to leverage familiar Salesforce features (reports, dashboards, list views, etc.) to manage and analyze logs.
2. **Platform Event Architecture**: The framework utilizes Salesforce Platform Events, enabling us to both log information AND throw exceptions - a critical capability that was virtually impossible in Salesforce before Platform Events. The "Log Entry Event Stream" tab allows real-time monitoring of these events.
3. **Comprehensive Information Capture**: Nebula Logger automatically captures extensive contextual data about the execution environment, including org details, user information, platform limits, and record data.
4. **Flexible Configuration**: The framework provides granular control over logging levels (ERROR, WARN, INFO, DEBUG, FINE, FINER, FINEST) that can be configured at the org, profile, or user level, allowing us to increase logging detail for specific teams or during new feature rollouts.
5. **Support for All Development Approaches**: With built-in support for Apex, Flow, and Lightning Components (both LWC and Aura), the framework accommodates our full development stack, enabling consistent logging practices across all automation types.
6. **Enhanced Data Security**: Data masking rules allow us to automatically protect sensitive information (like SSNs and credit card numbers) from being stored in logs, addressing compliance requirements.
7. **Scenario-Based Log Management**: The scenario feature allows us to categorize transactions and apply specific logging levels and retention policies to different types of operations.
8. **Transaction Tracking**: The ability to track related logs across asynchronous processes gives us visibility into complex operations that span multiple transactions.
9. **Extensibility via Plugins**: The plugin framework allows us to extend functionality without modifying core code, enabling integration with external systems like Slack and custom dashboards.
10. **Automatic Log Management**: Built-in purging capabilities help us manage data storage by automatically removing old logs based on configurable retention periods.
11. **Unlocked Package Deployment**: Available as an unlocked package, we can see and modify the underlying code if needed, providing greater flexibility for customization and security auditing.
12. **Open Source with Active Community**: As one of the most starred Salesforce logging repositories on GitHub, Nebula Logger continues to evolve and improve, ensuring we benefit from ongoing enhancements and refinements.

## Alternatives

Several alternatives to Nebula Logger exist for handling logging in Salesforce:

1. **Salesforce Native Debug Logs**: The platform's built-in debug logs provide basic logging functionality but have significant limitations including 24-hour retention, 20MB size limits, potential truncation, and limited reporting capabilities.
2. **Custom-Built Logging Solutions**: Organizations can build their own logging frameworks, but this requires significant development time and ongoing maintenance without the benefit of community support and continuous improvement.
3. **External Logging Services**: Solutions like Loggly or Rollbar can be integrated with Salesforce, but require additional infrastructure, often have per-seat licensing costs, and require developing integration points between Salesforce and external systems.
4. **AppExchange Logging Solutions**: Various paid logging solutions exist on the AppExchange, but many have ongoing subscription costs and may not provide the same level of flexibility and customization.

Nebula Logger provides the best balance of being a free, open-source solution with rich features specifically designed for the Salesforce platform while maintaining the flexibility to integrate with external systems through its plugin architecture when needed.
