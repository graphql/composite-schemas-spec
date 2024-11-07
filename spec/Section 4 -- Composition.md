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

#### Non-Null Input Fields cannot be inaccessible

**Error Code**

NON_NULL_INPUT_FIELD_IS_INACCESSIBLE

**Formal Specification**

- Let {fields} be the set of all fields of the input types
- For each {field} in {fields}:
  - If {field} is a non-null input field:
    - {field} must not be declared as `@inaccessible`

**Explanatory Text**

In a composed schema, a non-null input field must always be accessible. If it is
not, the field would require a value, yet the user would be unable to supply
one.

A valid case where a public input field references another public input type:

```graphql example
input Input1 {
  field1: String!
  field2: Input2
}

input Input2 {
  field3: String
}
```

Another valid case is where the field is not exposed in the composed schema:

```graphql example
input Input1 {
  field1: String!
  field2: Input2 @inaccessible
}

input Input2 @inaccessible {
  field3: String
}
```

An invalid case is when a non-null input field is inaccessible:

```graphql counter-example
input Input1 {
  field1: String! @inaccessible
  field2: Input2
}

input Input2 {
  field3: String
}
```

#### Input Fields cannot reference inaccessible type

**Error Code**

INPUT_FIELD_REFERENCES_INACCESSIBLE_TYPE

**Formal Specification**

- Let {fields} be the set of all fields of the input types
- For each {field} in {fields}:
  - If {field} is not declared as `@inaccessible`
    - Let {namedType} be the named type that {field} references
    - {namedType} must not be declared as `@inaccessible`

**Explanatory Text**

In a composed schema, a field within an input type must only reference types
that are exposed. This requirement guarantees that public types do not reference
inaccessible structures which are intended for internal use.

A valid case where a public input field references another public input type:

```graphql example
input Input1 {
  field1: String!
  field2: Input2
}

input Input2 {
  field3: String
}
```

Another valid case is where the field is not exposed in the composed schema:

```graphql example
input Input1 {
  field1: String!
  field2: Input2 @inaccessible
}

input Input2 @inaccessible {
  field3: String
}
```

An invalid case is when an input field references an inaccessible type:

```graphql counter-example
input Input1 {
  field1: String!
  field2: Input2!
}

input Input2 @inaccessible {
  field3: String
}
```

## Validate Satisfiability
