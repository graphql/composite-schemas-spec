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
  - Let {set} be the set in {typesByName} for {name}; if no such set exists, create it as an empty set.
  - Add {type} to {set}.
- For each {typesByName} as {name} and {types}:
  - Each pair of types in {types} must have the same kind.

**Explanatory Text**

The GraphQL Composite Schemas specification considers types with the same name
across source schemas as semantically equivalent and mergeable. Types that do
not share the same kind are considered non-mergeable.

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

```graphql example
scalar DateTime

scalar DateTime
```

```graphql counter-example
type User {
  id: ID!
  name: String!
  displayName: String!
  birthdate: String!
}

scalar User
```

```graphql counter-example
enum UserKind {
  A
  B
  C
}

scalar UserKind
```

### Merge

### Post Merge Validation

## Validate Satisfiability
