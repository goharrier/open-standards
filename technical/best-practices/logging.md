# Logging

The preferred logging framework for Salesforce development is [Nebula Logger](/technical/frameworks/nebula-logger-logging.md). It provides a structured and consistent way to handle logging across both Apex and Lightning Web Components (LWC).

**Avoid using `System.debug()` in Apex or `console.log()` in LWC for production-level logging.** These methods lack persistence, structured formatting, and centralized management capabilities essential for enterprise applications.

## How to Compose Good Log Messages

An effective log message should answer: Who, What, When, Where, Why, and How (when relevant).

[//]: # (to do: add examples)

## Logging Taxonomy

A well-defined logging taxonomy ensures consistency across your Salesforce org and enables effective observability. Structure your logs using these core dimensions:

### Severity Levels

Map business and technical events to appropriate severity levels:

| Level | Purpose | Example |
|------|--------|--------|
| `ERROR` | Unrecoverable failures, exceptions, integration failures | `Logger.error('Payment failed', e, paymentRecordId);` |
| `WARN` | Recoverable issues, deprecated usage, near-misses | `Logger.warn('Fallback API used due to timeout');` |
| `INFO` | Significant business events, milestones | `Logger.info('Order confirmed', order.Id);` |
| `DEBUG` | Detailed diagnostic data (disable in prod via settings) | `Logger.debug('Processing 50 records', records);` |
| `FINE`, `FINER`, `FINEST` | Verbose tracing (use sparingly) | `Logger.finer('Loop iteration: ' + i);` |

On production, set logging to WARN or ERROR to minimize noise. Use DEBUG or lower levels in development or troubleshooting scenarios.

### Log Categories

Organize logs by functional area to enable targeted filtering:

[//]: # (to do: how to use Nebula Logger scenarios Logger.setScenario)

### Contextual Metadata

Enrich logs with structured metadata for correlation and analysis.
Use the following methods from the `LogEntryBuilder` class to add context:

- `setRecord()` or `setRecordId()` when processing specific records.
- `setHttpRequestDetails()` and `setHttpResponseDetails()` for HTTP callouts.
- `setRestRequestDetails()` and `setRestResponseDetails()` for Apex REST web services.
- `addTag()` to include custom tags.

[//]: # (to do: explain when and how to use custom tags)
