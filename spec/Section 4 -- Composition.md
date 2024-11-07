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

**Severity**

ERROR

**Formal Specification**

- Let {typeNames} be the set of all output type names from all source schemas.
- For each {typeName} in {typeNames}
  - Let {types} be the set of all types with the name {typeName} from all source
    schemas.
  - Let {fieldNames} be the set of all field names from all {types}.
  - For each {fieldName} in {fieldNames}
    - Let {fields} be the set of all fields with the name {fieldName} from all
      {types}.
    - For each {field} in {fields}
      - Let {argumentNames} be the set of all argument names from all {fields}.
      - For each {argumentName} in {argumentNames}
        - Let {arguments} be the set of all arguments with the name
          {argumentName} from all {fields}.
        - For each pair of {argumentA} and {argumentB} in {arguments}
          - ArgumentsAreMergeable({argumentA}, {argumentB}) must be true.

ArgumentsAreMergeable(argumentA, argumentB):

- Let {typeA} be the type of {argumentA}
- Let {typeB} be the type of {argumentB}
- {SameInputTypeShape(typeA, typeB)} must be true.

**Explanatory Text**

Fields on mergeable objects or interfaces that have the same name are considered
semantically equivalent and mergeable when they have mergeable argument types.

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
