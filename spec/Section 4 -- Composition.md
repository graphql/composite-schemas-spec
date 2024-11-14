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

#### Enum Type Default Value Uses Inaccessible Value

**Error Code**

`ENUM_TYPE_DEFAULT_VALUE_INACCESSIBLE`

**Formal Specification**

- {ValidateArgumentDefaultValues()} must be true.
- {ValidateInputFieldDefaultValues()} must be true.

ValidateArgumentDefaultValues():

- Let {arguments} be all arguments of fields and directives across all source schemas
- For each {argument} in {arguments}
  - If {IsExposed(argument)} is true and has a default value:
    - Let {defaultValue} be the default value of {argument}
    - If not {ValidateDefaultValue(defaultValue)}
      - return false
- return true

ValidateInputFieldDefaultValues():

- Let {inputFields} be all input fields across all source schemas
- For each {inputField} in {inputFields}:
  - Let {type} be the type of {inputField}
  - If {IsExposed(inputField)} is true and {inputField} has a default value:
    - Let {defaultValue} be the default value of {inputField}
    - If {ValidateDefaultValue(defaultValue)} is false
      - return false
- return true

ValidateDefaultValue(defaultValue):

- If {defaultValue} is a ListValue:
  - For each {valueNode} in {defaultValue}:
    - If {ValidateDefaultValue(valueNode)} is false
      - return false
- If {defaultValue} is an ObjectValue:
  - Let {objectFields} be a list of all fields of {defaultValue}
  - Let {fields} be a list of all fields {objectFields} are referring to
  - For each {field} in {fields}:
    - If {IsExposed(field)} is false
      - return false
  - For each {objectField} in {objectFields}:
    - Let {value} be the value of {objectField}
    - return {ValidateDefaultValue(value)}
- If {defaultValue} is an EnumValue:
  - If {IsExposed(defaultValue)} is false
    - return false
- return true

**Explanatory Text**

This rule ensures that inaccessible enum values are not exposed in the composed schema through default values.
Output field arguments, input fields and directive arguments must only use enum values as their default value that is not annotated with the `@inaccessible` directive.

In this example the `FOO` value in the `Enum1` enum is not marked with
`@inaccessible`, hence it does not violate the rule.

```graphql
type Query {
  field(type: Enum1 = FOO): [Baz!]!
}

enum Enum1 {
  FOO
  BAR
}
```

The following example violates this rule because the default value for the field
`field` in type `Input1` references an enum value (`FOO`) that is marked as
`@inaccessible`.

```graphql counter-example
type Query {
  field(arg: Enum1 = FOO): [Baz!]!
}

input Input1 {
  field: Enum1 = FOO
}

directive @directive1(arg: Enum1 = FOO) on FIELD_DEFINITION

enum Enum1 {
  FOO @inaccessible
  BAR
}
```

```graphql counter-example
type Query {
  field(arg: Input1 = { field2: "ERROR" }): [Baz!]!
}

directive @directive1(arg: Input1 = { field2: "ERROR" }) on FIELD_DEFINITION

input Input1 {
  field1: String
  field2: String @inaccessible
}
```

### Merge

### Post Merge Validation

## Validate Satisfiability
