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

### Merge

### Post Merge Validation

#### Empty Merged Input Object Type

**Error Code**

INPUT_OBJECT_TYPE_EMPTY

**Formal Specification**

- Let {inputs} be the set of all input object in the merged schema
- For each {input} in {inputs}:
  - {IsInputObjectTypeEmpty(input)} must be false

IsInputObjectTypeEmpty(input):

- Let {fields} be a set of all input fields of {input}
- For each {field} in {fields}:
  - If {field} is not flagged as `@inaccessible`:
    - return false
- return true

**Explanatory Text**

When an input object type is defined in multiple source schemas, fields flagged as `@inaccessible` are not included in the merged input object type. 
If this process results in an input object type with no fields, the input object type is considered empty and invalid.

In the following example, the merged input object type `Input1` is valid. 

```graphql
input Input1 {
  field1: String @inaccessible
  field2: Int
}
```

This counter-example shows an invalid merged input object type. 
The `Input1` type is defined in two source schemas, but both `field1` and `field2` are flagged as
`@inaccessible` in at least on of the schemas:

```graphql counter-example
input Input1 {
  field1: String @inaccessible
  field2: Int @inaccessible
}
```

## Validate Satisfiability
