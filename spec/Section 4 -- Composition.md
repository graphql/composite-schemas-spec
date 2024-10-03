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

#### `@lookup` must not return a list 

**Error Code**

LOOKUP_MUST_NOT_RETURN_LIST

**Formal Specification**

- Let {fields} be the set of all field definitions annotated with `@lookup` in the schema.
- For each {field} in {fields}:
  - Let {type} be the return type of {field}.
  - {IsListType(type)} must be false.

IsListType(type):

- If {type} is a Non-Null type:
  - Let {innerType} be the inner type of {type}.
  - Return {IsSingleObjectType(innerType)}.
- Else if {type} is a List type:
  - Return true.
- Else:
  - Return false.

**Explanatory Text**

Fields annotated with the `@lookup` directive are intended to retrieve a single entity based on provided arguments. 
To avoid ambiguity in entity resolution, such fields must return a single object and not a list. 
This validation rule enforces that any field annotated with `@lookup` must have a return type that is NOT a list.

For example, the following usage is valid:

```graphql example
extend type Query {
  userById(id: ID!): User @lookup
}

type User {
  id: ID!
  name: String
}
```

In this example, `userById` returns a `User` object, satisfying the requirement.

This counter-example demonstrates an invalid usage:

```graphql counter-example
extend type Query {
  usersByRole(role: String!): [User] @lookup
}

type User {
  id: ID!
  name: String
}
```

Here, `usersByRole` returns a list of `User` objects, which violates the requirement that a `@lookup` field must return a single object.

### Merge

### Post Merge Validation

## Validate Satisfiability
