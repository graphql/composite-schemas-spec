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

#### Empty Merged Object Type

**Error Code**

`Empty_Merged_Object_Type`

**Formal Specification**

- Let {types} be the set of all object types across all source schemas
- For each {type} in {types}:
  - {IsObjectTypeEmpty(type)} must be false.

IsObjectTypeEmpty(type):

- Let {fields} be a set of all fields of all types with coordinate and kind
  {type} across all source schemas
- For each {field} in {fields}:
  - If {IsExposed(field)} is true
    - return false
- return true

**Explanatory Text**

For object types defined across multiple source schemas, the merged object type
is the superset of all fields defined in these source schemas. However, any
field marked with `@inaccessible` in any source schema is hidden and not
included in the merged object type. An object type with no fields, after
considering `@inaccessible` annotations, is considered empty and invalid.

In the following example, the merged object type `ObjectType1` is valid. It
includes all fields from both source schemas, with `field2` being hidden due to
the `@inaccessible` directive in one of the source schemas:

```graphql
type ObjectType1 {
  field1: String
  field2: Int @inaccessible
}

type ObjectType1 {
  field2: Int
  field3: Boolean
}
```

This counter-example demonstrates an invalid merged object type. In this case,
`ObjectType1` is defined in two source schemas, but all fields are marked as
`@inaccessible` in at least one of the source schemas, resulting in an empty
merged object type:

```graphql counter-example
type ObjectType1 {
  field1: String @inaccessible
  field2: Boolean
}

type ObjectType1 {
  field1: String
  field2: Boolean @inaccessible
}
```

## Validate Satisfiability
