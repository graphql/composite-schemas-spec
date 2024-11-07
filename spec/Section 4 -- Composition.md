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

**Severity** ERROR

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
  usersByIds(ids: [ID!]!): [User!] @lookup
}

type User {
  id: ID!
  name: String
}
```

Here, `usersByIds` returns a list of `User` objects, which violates the requirement that a `@lookup` field must return a single object.

//TODO explain the `@requires`

### Merge

### Post Merge Validation

#### Empty Merged Object Type

**Error Code**

`EMPTY_MERGED_OBJECT_TYPE`

**Severity** ERROR

**Formal Specification**

- Let {types} be the set of all object types across all source schemas
- For each {type} in {types}:
  - {IsObjectTypeEmpty(type)} must be false.

IsObjectTypeEmpty(type):

- If {type} has `@inaccessible` directive
- return false
- Let {fields} be a set of all fields in {type}
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

If the `@inaccessible` directive is applied to an object type itself, the entire
merged object type is excluded from the composite execution schema, and it is
not required to contain any fields.

```graphql
type ObjectType1 @inaccessible {
  field1: String
  field2: Int
}

type ObjectType1 {
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
