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

OBJECT_TYPE_EMPTY

**Formal Specification**

- Let {types} be the set of all object types in the merged schema.
- For each {type} in {types}:
  - {IsObjectTypeEmpty(type)} must be false.

IsObjectTypeEmpty(type):

- Let {fields} be a set of all fields of {type} 
- For each {field} in {fields}:
  - If {field} is not marked with `@inaccessible`:
    - return false
- return true

**Explanatory Text**

For object types defined across multiple source schemas, the merged object type is the superset of all fields defined in these schemas. 
However, any field marked with `@inaccessible` in any schema is hidden and not included in the merged object type. 
An object type with no fields, after considering `@inaccessible` markings, is considered empty and invalid.

In the following example, the merged object type `ObjectType1` is valid. 

```graphql
type ObjectType1 {
  field1: String
  field2: Int @inaccessible
}
```

This counter-example demonstrates an invalid merged object type. 
In this case, `ObjectType1` is defined in two source schemas, but all fields are marked as `@inaccessible` in at least one of the schemas, resulting in an empty merged object type:

```graphql counter-example
type ObjectType1 {
  field1: String @inaccessible
  field2: Boolean @inaccessible
}
```

## Validate Satisfiability
