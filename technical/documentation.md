# Documentation

## Diagrams

* [ ] System Architecture Diagram
* [ ] Data Flow Diagrams

## Design Decisions

Core design decisions are major technical decisions that impact the overall architecture of the platform and or a collection of features. These can be certain design principles that are being utilized from an integration point of view, sharing point of view or others.&#x20;

### Template: Design Decision X

**Problem Statement:** Short description of what the problem is we're designing a solution for.

**Research Insights:** Outline what research was conducted, what elements were included in the evaluation of the problem. What environmental factors play, for example: does the customer only have admins? Why would you then setup all triggers in Apex instead of focusing on Flows?

**Solution Hypothesis:** If we solve for the problem, what are the key things that will change and benefits that will be available to users / the company.

**Design Options:** a list of solution options that were evaluated. Each option should describe: WHAT, WHY, BENEFIT, DOWNSIDE (tradeoffs).

**Conclusion:** What has been decided and why.

## Feature Design Documentation

### Template: Feature Name

#### Functionality Design Goals

Replace with a brief description of the functionality we're attempting to build for, the purpose of the feature.

| Jira Story             | Link                  |
| ---------------------- | --------------------- |
| (List of Jira stories) | (corresponding links) |

#### Design Solutions

Replace with a contextual paragraph of how all the metadata components work together and in what sequence. (bullets and numbers are acceptable)&#x20;

#### Limitations and Tradeoffs&#x20;

Replace with a view into the design discussions, why we chose one thing versus another. Example: why we chose to use the APIs versus the manage package.

#### Components List

Replace with a table of all metadata components involved. No need to go into extreme detail unless required, use judgement. (page layouts, custom fields, etc.)

| Component                                   | Description                                                                     |
| ------------------------------------------- | ------------------------------------------------------------------------------- |
| RegisterNewOpportunity\_SF Flow             | Captures user input for new opportunity registration.                           |
| Validation Rules                            | Ensures data integrity with selective bypass for portal users.                  |
| Email Notification Flows                    | Manages email communications regarding opportunity submissions and assignments. |
| partnerLeadDetail, partnerOpportunityDetail | Custom components for detailed viewing of lead and opportunity records.         |
