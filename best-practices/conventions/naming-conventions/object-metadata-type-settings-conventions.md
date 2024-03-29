# Object / Metadata Type / Settings Conventions

All Naming Conventions Are [RFC 2119](https://www.ietf.org/rfc/rfc2119.txt) and [RFC 6919](https://tools.ietf.org/html/rfc6919) compliant.

1. All API names **MUST** be written in English, even when the label is in another language.
2. All API names **MUST** be written in PascalCase.
3. Objects **SHOULD NOT** contain an underscore in the objects name, except where explicitly defined otherwise in these conventions.
4. Objects generally **MUST (but you probably won't)** contain a description.
5. In all cases where the entire purpose of the object is not evident by reading the name, the object **MUST** contain a description.
6. Junction object API names **MUST** be singular (not pluralized) and contain the API names of **both** objects.
