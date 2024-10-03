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

#### `@lookup` Should Have Nullable Return Type

**Error Code**

LOOKUP_SHOULD_HAVE_NULLABLE_RETURN_TYPE

**Severity**

WARNING

**Formal Specification**

- Let {fields} be the set of all field definitions annotated with `@lookup` in the schema.
- For each {field} in {fields}:
  - Let {type} be the return type of {field}.
  - {type} must be a nullable type.

**Explanatory Text**

Fields annotated with the `@lookup` directive are intended to retrieve a single entity based on provided arguments. 
To properly handle cases where the requested entity does not exist, such fields should have a nullable return type. 
This allows the field to return `null` when an entity matching the provided criteria is not found, following the standard GraphQL practices for representing missing data.

In a distributed system, it is likely that some entities may not be found on other subgraphs, even when those subgraphs contribute fields to the type. 
Ensuring that `@lookup` fields have nullable return types also avoids GraphQL errors on subgraphs and prevents result erasure through non-null propagation. 
By allowing null to be returned when an entity is not found, the system can gracefully handle missing data without causing exceptions or unexpected behavior.

Ensuring that `@lookup` fields have nullable return types allows gateways to distinguish between cases where an entity is not found (receiving null) and other error conditions that may have to be propagated to the client.

For example, the following usage is recommended:

```graphql example
extend type Query {
  userById(id: ID!): User @lookup
}

type User {
  id: ID!
  name: String
}
```

In this example, `userById` returns a nullable `User` type, aligning with the recommendation.

This counter-example demonstrates a invalid usage:

```graphql counter-example
extend type Query {
  userById(id: ID!): User! @lookup
}

type User {
  id: ID!
  name: String
}
```

Here, `userById` returns a non-nullable `User!`, which does not align with the recommendation that a `@lookup` field should have a nullable return type.

### Merge

### Post Merge Validation

## Validate Satisfiability
