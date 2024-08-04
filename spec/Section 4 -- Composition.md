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

#### Output Field Types Mergeable

**Error Code**

OUTPUT_FIELD_TYPES_NOT_MERGEABLE

**Formal Specification**

- Let {fieldsByName} be a map of field lists where the key is the name of a
  field and the value is a list of fields from mergeable types from different
  subgraphs with the same name.
- for each {fields} in {fieldsByName}
  - {FieldsAreMergeable(fields)} must be true.

FieldsAreMergeable(fields):

- Given each pair of members {fieldA} and {fieldB} in {fields}:
  - Let {typeA} be the type of {fieldA}
  - Let {typeB} be the type of {fieldB}
  - {SameTypeShape(typeA, typeB)} must be true.

SameTypeShape(typeA, typeB):

- If {typeA} is Non-Null:
  - If {typeB} is nullable
    - Let {innerType} be the inner type of {typeA}
    - return SameTypeShape({innerType}, {typeB})
- If {typeB} is Non-Null:
  - If {typeA} is nullable
    - Let {innerType} be the inner type of {typeB}
    - return {SameTypeShape(typeA, innerType)}
- If {typeA} or {typeB} is List:
  - If {typeA} or {typeB} is not List, return false.
  - Let {innerTypeA} be the item type of {typeA}.
  - Let {innerTypeB} be the item type of {typeB}.
  - return {SameTypeShape(innerTypeA, innerTypeB)}
- If {typeA} and {typeB} are not of the same kind
  - return false
- If {typeA} and {typeB} do not have the same name
  - return false

**Explanatory Text**

Fields on mergeable objects or interfaces with that have the same name are
considered semantically equivalent and mergeable when they have a mergeable
field type.

Fields with the same type are mergeable.

```graphql example
type User {
  birthdate: String
}

type User {
  birthdate: String
}
```

Fields with different nullability are mergeable, resulting in merged field with
a nullable type.

```graphql example
type User {
  birthdate: String!
}

type User {
  birthdate: String
}
```

```graphql example
type User {
  tags: [String!]
}

type User {
  tags: [String]!
}

type User {
  tags: [String]
}
```

Fields are not mergeable if the named types are different in kind or name.

```graphql counter-example
type User {
  birthdate: String!
}

type User {
  birthdate: DateTime!
}
```

```graphql counter-example
type User {
  tags: [Tag]
}

type Tag {
  value: String
}

type User {
  tags: [Tag]
}

scalar Tag
```

### Merge

### Post Merge Validation

## Validate Satisfiability
