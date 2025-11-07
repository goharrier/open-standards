# Naming Conventions

Good naming conventions in software engineering are critical as they enhance readability and maintainability of the code. By adhering to a consistent set of rules for naming variables, functions, classes, and other entities, developers ensure that the code and configuration is self-explanatory, which is essential for team collaborations and future code revisions. Moreover, good naming practices reduce the learning curve for new team members and facilitate debugging and code analysis, ultimately leading to more robust and efficient software development.&#x20;

Follow SFDX naming conventions as a baseline, except where otherwise noted.

1. [Custom Objects, Custom Metadata Types, and Custom Settings](#custom-objects-custom-metadata-types-and-custom-settings)
2. [Custom Fields](https://wiki.sfxd.org/books/best-practices/page/general-conventions) (Follow SFXD Conventions)
3. [Validation Rules](https://wiki.sfxd.org/books/best-practices/chapter/validation-rule-conventions) (Follow SFXD Conventions)
4. [Workflow Rules](https://wiki.sfxd.org/books/best-practices/chapter/workflow-conventions) (Follow SFXD Conventions) :warning: <mark style="color:yellow;">DEPRECATED</mark>
5. [Custom Permissions](#custom-permissions)
6. [Profiles](#profiles)
7. [Permission Sets and Permission Set Groups](#permission-sets-and-permission-set-groups)
8. [Flows](https://wiki.sfxd.org/books/best-practices/page/flow-naming-conventions) (Follow SFXD Conventions)
9. [Named Credentials](#named-credentials)

All requirement keywords in these naming conventions use the terminology defined in [RFC 2119](https://datatracker.ietf.org/doc/html/rfc2119) extended by [RFC 6919](https://datatracker.ietf.org/doc/html/rfc6919).

## Custom Objects, Custom Metadata Types, and Custom Settings

1. All API names **MUST** be written in English, even when the label is in another language.
2. All API names **MUST** be written in PascalCase.
3. Objects **SHOULD NOT** contain an underscore in the objects name, except where explicitly defined otherwise in these conventions.
4. Objects generally **MUST (but you probably won't)** contain a description.
5. In all cases where the entire purpose of the object is not evident by reading the name, the object **MUST** contain a description.
6. Junction object API names **MUST** be singular (not pluralized) and contain the API names of **both** objects.

## Custom Permissions

1. All Permission API names **MUST** be written in English, even when the label is in another language.
2. All Permission API names **MUST** be written in PascalCase.
3. Permissions **SHOULD NOT** contain an underscore in the API name, except where explicitly defined otherwise in these conventions.

## Profiles

(to do)

## Permission Sets and Permission Set Groups

(to do)

## Named Credentials

(to do)
