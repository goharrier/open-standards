# Performance and Scalability

Salesforce is a multi-tenant platform — you fight for resources. Performance means staying under limits while delivering fast, scalable, predictable behavior.

## Good general rules

- Always bulkify
- Minimize database trips via Selectors + UoW
- Push heavy and retry-prone workloads async
- Cache when the risk of stale data < cost of re-querying
- Preserve transaction independence — avoid cascading failure chains

[//]: # (to do: include when to use @future vs. Queueable)

[//]: # (to do: include best practices for Batchable, when to use it, and when to prefer alternatives)

[//]: # (to do: include when to use Lightning Platform Cache, and best practices around it)

[//]: # (to do: include when to use Platform Events, and best practices around it)
