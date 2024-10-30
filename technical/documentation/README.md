# Documentation

## Diagrams

* [ ] [System Architecture Diagram](https://architect.salesforce.com/diagrams/framework/docs-implementation)
* [ ] [ERD](https://architect.salesforce.com/diagrams/framework/data-model-notation)
* [ ] Data Flow Diagrams
* [ ] UML Diagram

## **Design Decisions**

Core design decisions are significant technical choices that influence the overall architecture of a platform or a set of features. These decisions often adhere to specific design principles, facilitating better integration and information sharing among different parts of the system.

### **Template: Design Decision**

#### **Problem Statement**

A concise description of the issue that necessitates a design solution. This statement should highlight the critical problem we aim to solve.

#### **Research Insights**

Summarize the research conducted to understand the problem better.

Include:

* The components evaluated during the problem analysis.
* Any relevant environmental or contextual factors, such as the customer's internal capabilities. For instance, if a customer's team consists only of administrators, why would we opt to configure all triggers in Apex rather than leveraging Salesforce Flows? This section aims to shed light on the rationale behind certain design considerations based on research findings.

#### **Solution Hypothesis**

Describe the expected outcomes and benefits of addressing the problem. This section should articulate:

* The anticipated changes following the problem's resolution.
* The advantages these changes would bring to users or the organization as a whole.

#### **Design Options**

Present a list of potential solutions that were considered. For each option, detail the following:

* **WHAT:** A brief description of the solution.
* **WHY:** The rationale for considering this option.
* **BENEFIT:** The potential advantages of implementing the solution.
* **DOWNSIDE:** Any trade-offs or negative aspects associated with the solution. This part encourages a balanced view of each option, recognizing that most design choices involve compromises.

#### **Conclusion**

Summarize the chosen solution and justify the decision. This section should provide clarity on:

* The decision made regarding the design dilemma.
* The primary reasons behind selecting this particular solution over others.

## Feature Design Documentation

Design documentation is essential for individual features, not just for stories. It's crucial to design larger features thoroughly, rather than designing each story separately, to avoid a chaotic and poorly designed amalgamation of elements.

While smaller adjustments or enhancements to these features might not always necessitate changes in the documentation, their potential impact should still be assessed. Such documentation should be organized at the epic level to ensure clarity and cohesion.&#x20;

{% hint style="info" %}
When creating this feature-level documentation, it's useful to conduct a gut check: consider how often the documentation will need to be adjusted when new stories are introduced. If the answer is "often," then the documentation is too detailed at a low level.
{% endhint %}

### Template: Feature Name

#### **Functionality Design Goals**

Provide a succinct description of the functionality this feature aims to introduce. Outline the primary objectives and the need the feature addresses. This section sets the stage by explaining what the feature intends to achieve and its significance to users or the business.

| Jira Story             | Link                  |
| ---------------------- | --------------------- |
| (List of Jira stories) | (corresponding links) |

#### **Design Solutions**

Offer a detailed account of how various metadata components will function cohesively to realize the feature. Describe the sequence and interaction of these components, utilizing bullet points or numbers for clarity. This narrative should paint a clear picture of the feature's operational flow and how each element contributes to the overall functionality.

#### **Limitations and Tradeoffs**

Dive into the critical design discussions that shaped the final design choice. Highlight specific limitations encountered during the design process and the tradeoffs made to navigate these challenges. For instance, elaborate on the decision to utilize APIs over a managed package, detailing the reasoning and benefits of this choice versus the alternatives considered.

#### **Components List**

Provide a comprehensive table or list of all metadata components involved in the feature's design. While exhaustive detail isn't necessary for each component, employ discretion to decide when additional explanation is warranted. Essential components might include objects, main entry apex classes, lightning web components, and more.&#x20;

This list serves as a blueprint of the technical elements that constitute the feature.

| Component                | Description                                                                     |
| ------------------------ | ------------------------------------------------------------------------------- |
| Example\_SF Flow         | Captures user input for new X registration.                                     |
| Validation Rules         | Ensures data integrity with selective bypass for X users.                       |
| Email Notification Flows | Manages email communications regarding opportunity submissions and assignments. |
| componentDetail          | Custom lwc for detailed viewing of lead and opportunity records.                |

_Table: Example component list_
