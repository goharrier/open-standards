# Requirement definition

Requirements should be formatted in a consistent way and always contain 3 main elements:

* Requirement (User Story) definition itself (_As a_ x, _I want to be able to_ y, _so that_ z)
* Acceptance Criteria (written in [Gherkin syntax](https://cucumber.io/docs/gherkin/reference/): given, when, then)
* High level solution design notes

{% hint style="info" %}
We do not distinguish between the definitions of requirements in theoretical models, nor do we strictly adhere to 'the book' when it comes to Scrum, Agile, or Waterfall methodologies.
{% endhint %}

## AI-Assisted Requirement Definition

When working with AI coding assistants, well-structured requirements become even more critical. The format described on this page serves as the foundation for [spec-driven development](./spec-driven-development.md), which uses Open Spec's artifact-guided workflow to produce complete, consistent specifications that AI assistants can reliably implement.

See also:
- [Spec-Driven Development](./spec-driven-development.md) - Using Open Spec for AI-assisted requirement generation



### **1. User Story (Requirement) Definition**

> **Template Format**:
>
> **As a** \[user/role/persona], **I want** \[capability/functionality], **so that** \[benefit/value].

#### **Explanation**

1. **Who**: Identify the primary user or role that will benefit from the feature. It could be a job function (e.g., “As a project manager”) or a type of end user (e.g., “As a customer”).
2. **What**: Clearly state the functionality or capability the user needs (e.g., “I want to export data to CSV”).
3. **Why**: Explain the value or benefit of having that capability (e.g., “so that I can analyze it in a spreadsheet”).

#### **Good Example**

> **As a** Customer Support Agent,\
> **I want** to filter tickets by their status,\
> **so that** I can quickly prioritize unresolved issues.

* **Who**: Customer Support Agent
* **What**: Ability to filter tickets by status
* **Why**: To prioritize unresolved issues and work more efficiently

#### **Bad Example**

> **As a** user,\
> **I want** the system to be better,\
> **so that** it’s easier to use.

* **Vagueness**: Does not specify the user persona clearly (simply says “user”).
* **Lack of clarity**: “System to be better” is not a concrete requirement.
* **Unclear benefit**: “So that it’s easier to use” is too generic and gives no measurable outcome.

***

### **2. Acceptance Criteria (Gherkin Syntax)**

> **Template Format**:
>
> **Scenario**: \[Meaningful title or label]\
> **Given** \[initial context/conditions],\
> **When** \[action/event],\
> **Then** \[expected outcome/result].

#### **Explanation**

1. **Scenario Title**: Give each scenario a descriptive name that quickly conveys its purpose.
2. **Given**: Pre-conditions or context required before the user takes an action.
3. **When**: The action the user takes or the event that triggers a behavior.
4. **Then**: The outcome or the result that must be achieved to consider this scenario successful.

You can have multiple **Given–When–Then** sequences within a single scenario if needed, or multiple scenarios under the same story to cover different permutations.

#### **Good Example**

> **Scenario**: Filter tickets by status\
> **Given** I am logged in as a Customer Support Agent\
> **And** I have at least one ticket with the status “Open” and one ticket with the status “Closed”\
> **When** I navigate to the ticket dashboard\
> **And** I apply a filter for “Open” tickets\
> **Then** I should only see tickets with status “Open” in the list\
> **And** I should not see any tickets with the status “Closed”

* **Clear context** (“I am logged in as a Customer Support Agent” and tickets exist).
* **Distinct action** (“apply a filter for ‘Open’ tickets”).
* **Valid expected outcome** (“only see tickets with status ‘Open’”).

#### **Bad Example**

> **Scenario**: Filter\
> **Given** the system is open,\
> **When** I filter,\
> **Then** I should see results.

* **Vague context**: “the system is open” is not precise—where is the user, what page or screen is active?
* **Incomplete**: “I filter” does not describe how or what filter is being applied.
* **No clear outcome**: “I should see results” is too generic.

***

### **3. High-Level Solution Design Notes**

> **Template Format**:
>
> * **Technical Considerations**: Summarize any major technical components, APIs, or integrations needed.
> * **Constraints or Limitations**: Mention performance constraints, platform constraints, or known limitations.
> * **UX/UI Considerations**: Briefly note any specific UI elements or design guidelines that must be followed.
> * **Dependencies**: List any upstream or downstream dependencies that might affect this user story (e.g., other microservices, external data sources, or features).

#### **Explanation**

1. **Technical considerations**: Identify if new components, services, or data structures are needed.
2. **Constraints or limitations**: E.g., maximum number of records, device constraints, security, etc.
3. **UX/UI considerations**: Include a rough sketch, wireframe reference, or important design elements if necessary.
4. **Dependencies**: Call out any reliant features or external tools.

#### **Good Example**

> * **Technical Considerations**:
>   * Implement an API endpoint `/tickets/filter` to handle filtering logic on the backend.
>   * Use indexed queries in the database to improve filter performance.
> * **Constraints**:
>   * Must support up to 10,000 tickets in the list.
>   * Filtering should respond within 2 seconds.
> * **UX/UI Considerations**:
>   * The filter dropdown should display all possible status options (Open, In Progress, Closed).
>   * The default state of the filter is set to “All Statuses.”
> * **Dependencies**:
>   * The user login and authentication service must be implemented before testing this feature.
>   * Any changes to ticket statuses in other services should trigger an update in our ticketing database.

* **Concise yet informative**: It highlights the back-end endpoint and performance requirements without going into overly detailed technical documentation.
* **Clear constraints**: Performance metrics and capacity.
* **User experience**: The dropdown status filter is specified.
* **Dependencies**: Auth service and database updates are mentioned.

#### **Bad Example**

> * **Technical Stuff**: We’ll do something with the database.
> * **Constraints**: Shouldn’t be too slow.
> * **Design**: Make it look nice.
> * **Dependencies**: Not sure yet.

* **Lacks details**: “Do something with the database” is not actionable or specific.
* **No performance expectations**: “Shouldn’t be too slow” is too vague.
* **No clarity**: “Make it look nice” does not reference any design guidelines.
* **No identified dependencies**: “Not sure yet” is not helpful.

***

## **Complete Template**

Below is a **copy-paste-ready template** you can use for your own user stories. Simply fill in the sections with details specific to your requirement.

***

```
## User Story

**As a** [user persona or role],  
**I want** [capability or functionality],  
**so that** [benefit or value].

## Acceptance Criteria

### Scenario 1: [Descriptive Title]
**Given** [initial context/conditions],  
**And** [additional context if any],  
**When** [action the user or system takes],  
**And** [any additional event if required],  
**Then** [expected outcome],  
**And** [any additional outcome if required].

### Scenario 2: [Additional scenario title, if necessary]
[Repeat Gherkin steps as needed]

## High-Level Solution Design

- **Technical Considerations**:
  - [Brief note on approach, services, APIs, data schemas, etc.]
- **Constraints or Limitations**:
  - [Performance, device constraints, data volume, etc.]
- **UX/UI Considerations**:
  - [Any relevant design guidelines, references to wireframes, or key UI elements]
- **Dependencies**:
  - [Other features, services, or external components that impact this user story]
```

***

### **How to Use This Template Effectively**

1. **Keep the “why” clear**: The user story must focus on the user’s goal and the value it provides.
2. **Be specific in acceptance criteria**: Use **Given-When-Then** to detail all conditions, actions, and outcomes, ensuring no ambiguity.
3. **Keep solution notes high-level**: Details in this section should aid in understanding and planning without becoming full technical designs.
4. **Review often**: Regularly revisit and refine the story, criteria, and design notes to ensure they remain accurate and aligned with the project goals.

Following these guidelines will help ensure your requirements are well-defined, testable, and provide clear value to stakeholders.
