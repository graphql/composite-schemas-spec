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

#### Semantic Equivalence of Types

**Error Code**

`TYPE_KIND_NOT_MERGEABLE`

**Formal Specification**

- Let {typesByName} be an empty unordered map of sets.
- For each named type {type} across all source schemas:
  - Let {name} be the name of {type}.
  - Let {set} be the set in {typesByName} for {name}; if no such set exists,
    create it as an empty set.
  - Add {type} to {set}.
- For each {typesByName} as {name} and {types}:
  - Each pair of types in {types} must have the same kind.

**Explanatory Text**

The GraphQL Composite Schemas specification considers types with the same name
across source schemas as semantically equivalent and mergeable.

For the schema composition process to be able to merge types with the
same name, the types must have the same kind. In this example we have two types
called `User` which are both object types and are mergeable.

```graphql example
type User {
  id: ID!
  name: String!
  displayName: String!
  birthdate: String!
}

type User {
  id: ID!
  name: String!
  reviews: [Review!]
}
```

However, if the second type were a scalar type, the types would not be
mergeable as they have different kinds.

```graphql counter-example
type User {
  id: ID!
  name: String!
  displayName: String!
  birthdate: String!
}

scalar User
```

All types with the same name must have the same kind to be mergeable.

```graphql example
scalar User

scalar User
```

### Merge

### Post Merge Validation

## Validate Satisfiability
