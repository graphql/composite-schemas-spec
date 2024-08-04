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

#### Output Field Argument Types Mergeable

**Error Code**

OUTPUT_FIELD_ARGUMENT_TYPES_NOT_MERGEABLE

**Formal Specification**

- Let {fieldsByName} be a map of field lists where the key is the name of a
  field and the value is a list of fields from mergeable types from different
  subgraphs with the same name.
- for each {fields} in {fieldsByName}
  - if {FieldsInSetCanMerge(fields)} must be true.

FieldsAreMergeable(fields):

- Given each pair of members {fieldA} and {fieldB} in {fields}:
  - {ArgumentsAreMergeable(fieldA, fieldB)} must be true.

ArgumentsAreMergeable(fieldA, fieldB):

- Given each pair of arguments {argumentA} and {argumentB} in {fieldA} and
  {fieldB}:
  - Let {typeA} be the type of {argumentA}
  - Let {typeB} be the type of {argumentB}
  - {SameInputTypeShape(typeA, typeB)} must be true.

**Explanatory Text**

Fields on mergeable objects or interfaces that have the same name are considered semantically 
equivalent and mergeable when they have a mergeable argument types.

```graphql example
type User {
  field(argument: String): String
}

type User {
  field(argument: String): String
}
```

Arguments that differ on nullability of an argument type are mergeable.

```graphql example
type User {
  field(argument: String!): String
}

type User {
  field(argument: String): String
}
```

```graphql example
type User {
  field(argument: [String!]): String
}

type User {
  field(argument: [String]!): String
}

type User {
  field(argument: [String]): String
}
```

Arguments are not mergeable if the named types are different in kind or name.

```graphql counter-example
type User {
  field(argument: String!): String
}

type User {
  field(argument: DateTime): String
}
```

```graphql counter-example
type User {
  field(argument: [String]): String
}

type User {
  field(argument: [DateTime]): String
}
```

### Merge

### Post Merge Validation

## Validate Satisfiability
