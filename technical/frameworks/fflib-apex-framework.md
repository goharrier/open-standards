# fflib - Apex Framework

### Links

| Description        | Url                                                                                                                                                                                                                                                                                             |
| ------------------ | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| main               | <p><a href="https://github.com/apex-enterprise-patterns/fflib-apex-mocks">https://github.com/apex-enterprise-patterns - fflib-apex-mocks</a><br><a href="https://github.com/apex-enterprise-patterns/fflib-apex-common">https://github.com/apex-enterprise-patterns - fflib-apex-common</a></p> |
| extensions         | [https://github.com/wimvelzeboer/fflib-apex-extensions](https://github.com/wimvelzeboer/fflib-apex-extensions)                                                                                                                                                                                  |
| template generator | [https://github.com/wimvelzeboer/fflib-templates](https://github.com/wimvelzeboer/fflib-templates) _(beta version)_                                                                                                                                                                             |

### Use case

fflib addresses fundamental challenges that plague Salesforce development at scale:

**1. Inconsistent Implementation Patterns**\
Without a framework, every developer implements solutions differently. One developer puts business logic in triggers, another in helper classes, a third creates utility methods. This inconsistency makes code reviews painful, onboarding slow, and maintenance a nightmare. fflib enforces a consistent, opinionated structure similar to Spring Framework in Java - everyone knows where to find and place specific types of logic.

**2. Testing Bottlenecks**\
Traditional Salesforce testing requires extensive data factories and real database operations. Tests take forever to run, often timeout, and break when validation rules change. fflib's mocking capabilities let you test business logic in isolation without touching the database, reducing test execution time from minutes to seconds.

**3. God Classes and Spaghetti Code**\
We've all seen them - 3000-line trigger handlers where business logic, database queries, and field updates are tangled together. fflib forces separation through its layer architecture:

* **Selectors** handle all SOQL queries
* **Domains** encapsulate record-level business logic and validation
* **Services** orchestrate complex business processes
* **Unit of Work** manages DML operations and transaction control

This separation makes code genuinely testable, reusable, and maintainable.

**4. Lack of Dependency Injection**\
Salesforce's static nature makes it hard to swap implementations for testing or different contexts. fflib provides dependency injection patterns that let you inject mock implementations during testing or different implementations based on configuration.

### Rationale

**Why we chose fflib (the honest take):**

**The Good:**

1. **Enforced Consistency** - fflib is opinionated, and that's its strength. It forces developers to think and structure code in a specific way. Once you learn the pattern, you can jump into any fflib codebase and immediately understand the architecture.
2. **Professional Software Engineering Practices** - It brings enterprise patterns (Service Layer, Repository, Unit of Work) to Salesforce. If you've worked with Spring, .NET, or similar frameworks, fflib feels familiar and natural.
3. **Genuine Testability** - With fflib-apex-mocks, you can write fast, isolated unit tests. We're talking about 90%+ code coverage with tests that run in seconds, not minutes. This alone justifies the framework for teams doing continuous integration.
4. **Scalable Architecture** - As your org grows from 10 to 1000 classes, fflib's structure prevents the codebase from becoming unmaintainable. You always know where to add new features.

**The Trade-offs (let's be real):**

1. **Steep Learning Curve** - If you're a declarative developer or self-taught coder without formal software engineering background, fflib will be challenging. Concepts like dependency injection, mocking, and layer separation require significant investment to understand and apply effectively.
2. **Performance Overhead** - fflib adds CPU time and memory consumption through its abstraction layers and boilerplate. For simple operations, you're loading multiple framework classes. In CPU-critical contexts, this matters.
3. **All-or-Nothing Commitment** - Half-hearted fflib adoption is worse than no fflib. If you use the framework but ignore its patterns, you get all the overhead with none of the benefits - slower code that's still poorly structured.
4. **Team Buy-in Required** - Everyone needs to understand and follow the patterns. One developer doing their own thing can undermine the entire architecture.

**Our Verdict:**\
For teams building enterprise-scale Salesforce applications with multiple developers, the benefits far outweigh the costs. The initial learning investment pays dividends in maintainability, testability, and developer productivity. However, for small orgs with simple requirements or teams without software engineering experience, the overhead might not be justified.

### Alternatives

| Approach                                                           | When It Works                                                             | When It Doesn't                                                             | Our Take                                                                                 |
| ------------------------------------------------------------------ | ------------------------------------------------------------------------- | --------------------------------------------------------------------------- | ---------------------------------------------------------------------------------------- |
| **No Framework (Vanilla Apex)**                                    | Small orgs, simple requirements, single developer                         | Multiple developers, complex business logic, need for extensive testing     | Works until it doesn't. Every org starts here, most regret not adopting structure sooner |
| **Trigger Handler Frameworks** (Trigger Framework by Kevin O'Hara) | Want minimal overhead, focus only on trigger management                   | Need service layer patterns, dependency injection, mocking                  | Good stepping stone but doesn't address broader architectural needs                      |
| **Custom Framework**                                               | Unique requirements, full control needed, experienced architect available | Most teams, ongoing maintenance burden, lack of community support           | Unless you have specific needs fflib doesn't meet, you're reinventing the wheel          |
| **AT4DX** (Salesforce Labs)                                        | Want Salesforce-official patterns, lighter weight than fflib              | Need comprehensive mocking, extensive community resources                   | Newer, less mature, smaller community. Worth watching but not yet proven at scale        |
| **Force-DI**                                                       | Only need dependency injection, want minimal framework                    | Need full architectural patterns, service layers, domain logic organization | Good for specific DI needs but doesn't provide complete architecture                     |

### Implementation Guidance

**Critical Success Factors:**

1. **Be Strict or Don't Bother**\
   We enforce layer separation strictly. No shortcuts, no "just this once" exceptions. The moment you let business logic creep into Selectors or queries into Services, you've undermined the entire pattern. Either commit fully or use something simpler.
2. **Start with the Team's Strongest Developer**\
   Have your most experienced software engineer implement the first few features using fflib. They'll establish patterns others can follow. Don't let everyone figure it out independently - you'll end up with different interpretations of the same patterns.
3. **Invest in Learning**\
   Budget 2-4 weeks for developers new to enterprise patterns to become productive with fflib. For experienced Java/.NET developers, it's faster. For Salesforce-only developers, it's a significant paradigm shift. Consider:\n - Internal workshops on DI, mocking, and layer architecture\n - Pair programming during initial implementation\n - Code review checklists specific to fflib patterns\n - A reference implementation in your codebase
4. **Mock Everything, Test Fast**\
   The framework's value emerges when you embrace mocking. If you're still writing tests that insert records, you're missing the point. Every test should run in milliseconds, not seconds. Learn fflib-apex-mocks deeply - it's as important as the framework itself.
5. **Layer Responsibilities (Our Rules):**
   1. **Selectors**: SOQL only. No business logic. Not even "simple" field checks
   2. **Domains**: Single-object business logic, validation, defaulting. No cross-object operations
   3. **Services**: Multi-object orchestration, transaction control. No direct SOQL
   4. **Unit of Work**: DML operations only. Register operations, don't execute them directly
6. **Performance Considerations** \
   Accept that fflib adds overhead. In most cases, the maintainability gain justifies the performance cost. However:
   1. Profile critical code paths
   2. Consider bypassing the framework for high-volume batch operations
   3. Use lazy initialization for framework components
   4. Don't over-engineer simple operations
7. **Common Pitfalls to Avoid:**
   1. Mixing fflib and non-fflib patterns in the same module
   2. Creating anemic domains (domains with no behavior)
   3. Skipping the Service layer for "simple" operations
   4. Using Unit of Work inconsistently
   5. Not mocking in tests "because it's easier to use real data"
8. **When to Break the Rules:** \
   Almost never. The framework's value comes from consistency. The only exceptions we make:
   1. Platform event triggers (often need immediate DML)
   2. High-volume batch processing (performance critical)
   3. Simple configuration/metadata operations (overhead not justified)
