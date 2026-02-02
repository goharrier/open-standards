# AI Coding Agents

{% hint style="info" %}
**Rapidly Evolving Ecosystem**: AI coding tools and capabilities are evolving at an unprecedented pace. This documentation reflects our approach as of early 2026. The tools, features, and best practices in this space change frequently. Always refer to the official [Claude Code documentation](https://github.com/anthropics/claude-code) for the most current capabilities and implementation guidance.
{% endhint %}

## Why AI Coding Agents Matter

Traditional development relies on developers manually writing every line of code, searching documentation, and orchestrating complex workflows across multiple files and systems. AI coding agents fundamentally change this dynamic by acting as **intelligent development partners** that understand context, generate code aligned with established patterns, and automate repetitive tasks.

However, not all AI coding tools are created equal. The difference between a basic code completion tool and a sophisticated coding agent is the difference between autocomplete and a senior developer who understands your architecture, follows your standards, and can orchestrate complex multi-step changes.

This is why **tool choice matters**. The wrong tool creates as many problems as it solves—generating code that violates your patterns, missing critical context, or requiring constant manual correction. The right tool amplifies your team's effectiveness while maintaining code quality and architectural consistency.

## Why We Standardized on Claude Code

We needed a coding agent that could:
- **Understand and follow our established patterns** (fflib, Nebula Logger, our naming conventions)
- **Handle complex, multi-file changes** with full context awareness
- **Be customizable** to our specific Salesforce development workflow
- **Integrate with our standards** rather than forcing us to adapt to the tool

[Claude Code](https://github.com/anthropics/claude-code) meets these requirements through three critical capabilities:

### 1. Skills: Custom Workflows for Our Standards

Claude Code's skill system allows us to create **custom workflows that encode our development standards**. Rather than repeatedly explaining "follow the fflib pattern" or "use our naming conventions," we build skills that:

- Automatically apply our Apex style guide
- Generate code following fflib patterns, including fflib extensions (AT4DX, etc.)
- Create test classes with fflib_ApexMocks structure
- Enforce naming conventions for all types of objects, fields, and classes
- Apply SOLID principles in design and implementation
- Follow enterprise design patterns appropriate to the context

**Why this matters**: This is fundamentally different from simple template-based code generation. Templates produce rigid, one-size-fits-all code. Skills **understand context and apply principles**, generating code that respects SOLID design, uses appropriate patterns for the specific use case, and maintains architectural consistency.

A junior developer gets the same pattern-compliant, well-architected code as a senior developer because the standards are **encoded in the workflow**, not dependent on individual knowledge.

This aligns with our philosophy from [spec-driven development](../functional/requirement-definition/spec-driven-development.md)—**structure and guidance produce better results than relying on individual execution**.

### 2. Plugins: Orchestrating Agents and Skills for Complete Workflows

Claude Code's plugin architecture allows us to **orchestrate multiple agents and skills into cohesive, process-focused workflows**. Plugins enable us to:

- Create **hyper-focused capabilities** for specific contexts (frontend development, backend services, bug fixing, refactoring)
- **Combine specialized agents and skills** into end-to-end development lifecycles
- Follow **specific processes** tailored to different types of work
- Build **composable workflows** where focused elements work together

For example, a "backend service development" plugin might orchestrate:
1. A domain modeling skill (applying fflib patterns and SOLID principles)
2. A service layer agent (generating business logic with proper separation of concerns)
3. A selector skill (creating data access following fflib_SObjectSelector patterns)
4. A testing agent (generating comprehensive tests with mocking)

Each component is hyper-focused on one thing. The plugin combines them into a **complete backend development workflow** that follows our standards end-to-end.

Similarly, a "bug fixing" plugin orchestrates different capabilities—debugging agents, code analysis skills, test generation—tailored specifically to the bug fixing process rather than new feature development.

### 3. Agents: Multi-Step Orchestration

The most powerful capability is Claude Code's agent system—**autonomous execution of multi-step workflows**:

- **Refactoring agents**: Analyze code, identify patterns, propose and execute refactorings across multiple files
- **Testing agents**: Generate comprehensive test coverage following our mocking patterns
- **Documentation agents**: Create technical documentation aligned with our standards
- **Review agents**: Analyze code for security vulnerabilities, governor limit violations, and pattern compliance

**Why this matters**: Complex tasks—like refactoring a service class to follow Domain-Driven Design patterns—require understanding context across multiple files, making coordinated changes, and ensuring nothing breaks. Agents orchestrate these multi-step workflows autonomously, maintaining context and following established patterns throughout.

This moves beyond "code completion" to **intelligent development orchestration**.

## AI Orchestration with Claude Code

Claude Code's architecture supports **complete AI-assisted development workflows** through the coordination of skills, agents, and plugins:

| Stage | Capability | Purpose |
|-------|-----------|---------|
| **Requirement Definition** | Open Spec skill | Generate structured specifications with Gherkin scenarios |
| **Code Generation** | Skills + plugins | Generate pattern-compliant code from specifications |
| **Multi-File Orchestration** | Agents | Coordinate changes across services, domains, selectors, and tests |
| **Review and Testing** | Review/testing agents | Verify code quality, security, and test coverage |
| **Process Workflows** | Plugins | Orchestrate end-to-end workflows (frontend, backend, bug fixing) |

This orchestration ensures that:
1. Requirements are structured and complete (Open Spec skill)
2. Code follows established patterns (skills)
3. Complex changes are coordinated correctly (agents)
4. Complete workflows are automated (plugins)
5. Quality is verified systematically (review/testing agents)

## Alignment with Our Development Standards

Our use of Claude Code aligns with our core development principles:

### Pattern Compliance

We've invested heavily in establishing patterns—[fflib framework](./frameworks/fflib-apex-framework.md), [enterprise design patterns](./architecture-and-design-patterns/README.md), [JSON storage patterns](./architecture-and-design-patterns/json-field-storage-pattern.md). Claude Code's skills ensure AI-generated code **respects these patterns** rather than introducing inconsistent implementations.

### Traceability

Just as [spec-driven development](../functional/requirement-definition/spec-driven-development.md) provides traceability from requirements to code, Claude Code agents maintain traceability from specification scenarios to implementation. Each Gherkin scenario maps to specific code, and agents ensure complete coverage.

### Knowledge Preservation

Claude Code's skills and plugins capture **institutional knowledge as executable workflows**. Rather than relying on individual developers to remember patterns, the workflows encode:
- How to implement features following our standards
- Which patterns apply to specific scenarios
- The sequence of steps for complex processes

When a developer works on a feature, they're guided by these encoded workflows rather than having to reconstruct best practices from documentation.

### Pragmatic Complexity

We apply the right level of tooling for the task. Not every change requires agent orchestration—sometimes direct coding is faster. But for complex refactorings, new feature development, or cross-cutting changes, the orchestration capabilities provide significant value.

This matches our philosophy: **Apply the level of tooling that serves the goal, not tooling for its own sake**.

## When to Use AI Coding Agents

### High Value for AI Assistance

- **New feature implementation**: Generating services, domains, selectors, and tests following established patterns
- **Refactoring to patterns**: Converting ad-hoc code to structured patterns (e.g., introducing Unit of Work)
- **Test generation**: Creating comprehensive test coverage with proper mocking
- **Cross-cutting changes**: Updates that touch multiple layers (API changes propagating through stack)
- **Documentation generation**: Technical documentation aligned with our standards

### Lower Value for AI Assistance

- **Exploratory coding**: When you're still figuring out the approach
- **Critical security logic**: Where human review is paramount
- **Novel patterns**: When establishing new patterns rather than following existing ones
- **Emergency hotfixes**: When speed trumps pattern compliance

{% hint style="info" %}
**Pragmatic Application**: AI coding agents are powerful tools, not silver bullets. Use them where they provide clear value—pattern-compliant code generation, multi-file orchestration, and maintaining consistency. Continue using traditional development where it's more effective—exploration, novel problem-solving, and critical security logic.
{% endhint %}

## Why These Tools Instead of Alternatives

The AI coding agent landscape includes many options:

**GitHub Copilot** excels at inline code completion but lacks the structured skill system, agent orchestration, and Salesforce-specific understanding that Claude Code provides. It's optimized for "next line prediction" rather than "multi-file pattern-compliant implementation."

**Cursor / Windsurf** offer AI-powered IDEs with strong code generation capabilities but are generic development tools. They lack the customization (skills), orchestration (agents), and workflow encoding (plugins) required for enterprise Salesforce development.

**Generic LLM chat interfaces** (ChatGPT, Claude web) provide AI assistance but lack codebase context, can't execute changes directly, and don't maintain state across sessions. They're useful for isolated questions but ineffective for complex development workflows.

**Claude Code** uniquely provides:
- **Customizable workflows** (skills) that encode our standards
- **Autonomous orchestration** (agents) for multi-step tasks
- **Process-focused plugins** that combine hyper-focused capabilities into complete workflows
- **Integration with spec-driven development** for end-to-end AI-assisted workflows
- **Extensibility** to evolve with our practices and standards

For our context—enterprise Salesforce development with established patterns, complex multi-package architectures, and requirements for traceability—Claude Code provides the right balance of power and control.

## Adopting AI Coding Agents

If you're adopting Claude Code:

1. **Start with our established patterns**: Review our [best practices](./best-practices/README.md), [frameworks](./frameworks/README.md), and [architecture patterns](./architecture-and-design-patterns/README.md) to understand the standards your skills should enforce

2. **Learn Claude Code fundamentals**: Understand the [skill system](https://github.com/anthropics/claude-code#skills), agent capabilities, and plugin architecture before building custom workflows

3. **Begin with simple skills**: Create a skill for a single pattern (e.g., "generate a selector following fflib_SObjectSelector") before building complex multi-step workflows

4. **Build process-focused plugins**: Combine skills and agents into plugins for specific workflows (frontend development, backend services, bug fixing)

5. **Integrate with spec-driven development**: Connect Open Spec requirement generation with Claude Code implementation to experience the full AI-assisted workflow

6. **Create team workflows**: Build shared skills and plugins that encode your team's patterns, making them reusable across developers

The goal is not to replace developer judgment—it's to **amplify developer effectiveness** by automating pattern application, orchestrating complex changes, and maintaining consistency across a large codebase.

## Resources

- [Claude Code Repository](https://github.com/anthropics/claude-code) - Official documentation and skill development guides
- [Our Spec-Driven Development Standard](../functional/requirement-definition/spec-driven-development.md) - How AI coding agents integrate with requirement generation
- [fflib Apex Framework](./frameworks/fflib-apex-framework.md) - Patterns that skills should enforce
- [Best Practices](./best-practices/README.md) - Standards for AI-generated code
- [Architecture and Design Patterns](./architecture-and-design-patterns/README.md) - Enterprise patterns to encode in workflows
