# Schema Composition

The schema composition describes the process of merging multiple source schemas
into a single GraphQL schema, known as the _composite execution schema_, which
is a valid GraphQL schema annotated with execution directives. This composite
execution schema is the output of the schema composition process. The schema
composition process is divided into four major algorithms: **Validate Source
Schema**, **Merge Source Schema**, and **Validate Satisfiability**, which are
run in sequence to produce the composite execution schema.

## Validate Source Schema

## Merge Source Schemas

### Pre Merge Validation

#### Enum Type Values Must Be The Same Across Source Schemas

**Error Code**

ENUM_VALUES_MUST_BE_THE_SAME_ACROSS_SCHEMAS

**Formal Specification**

- Let {enumNames} be the set of all enum type names across all source schemas.
- For each {enumName} in {enumNames}:
  - Let {enums} be the list of all enum types from different source schemas with the name {enumName}.
  - {EnumsAreMergeable(enums)} must be true.

EnumsAreMergeable(enums):

- Let {inaccessibleValues} be the set of values that are declared as `@inaccessible` in {enums}.
- Let {requriedValues} be the set of values in {enums} that are not in {inaccessibleValues}.
- For each {enum} in {enums}
  - Let {enumValues} be the set of all values of {enum} that are not in {inaccessibleValues}.
  - {requriedValues} must be equal to {enumValues}

**Explanatory Text**

This rule ensures that enum types with the same name across different source schemas in a composite schema have identical sets of values. 
Enums, must be consistent across source schemas to avoid conflicts and ambiguities in the composite schema.

When an enum is defined with differing values, it can lead to confusion and errors in query execution. 
For instance, a value valid in one schema might be passed to another where it's unrecognized, leading to unexpected behavior or failures. 
This rule prevents such inconsistencies by enforcing that all instances of the same named enum across schemas have an exact match in their values.

In this example, both source schemas define `Enum1` with the same value `BAR`, satisfying the rule:

```graphql example
enum Enum1 {
  BAR
}

enum Enum1 {
  BAR
}
```

Here, the two definitions of `Enum1` have different values (`BAR` and `Baz`), violating the rule:

```graphql counter-example
enum Enum1 {
  BAR
}

enum Enum1 {
  Baz
}
```

Here, the two definitions of `Enum1` have shared values and additional values declared as `@inaccessible`, satisfying the rule:

```graphql example
enum Enum1 {
  BAR
  BAZ @inaccessible
}

enum Enum1 {
  BAR
}
``` 

### Merge

### Post Merge Validation

## Validate Satisfiability
