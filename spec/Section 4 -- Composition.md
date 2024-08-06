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

#### Input Field Types mergeable

**Error Code**

`INPUT_FIELD_TYPES_MERGABLE`

**Formal Specification**

- Let {fieldsByName} be a map of field lists where the key is the name of a
  field and the value is a list of fields from mergeable input types from
  different source schemas with the same name.
- For each {fields} in {fieldsByName}:
  - if {InputFieldsAremergeable(fields)} must be true.

InputFieldsAremergeable(fields):

- Given each pair of members {fieldA} and {fieldB} in {fields}:
  - Let {typeA} be the type of {fieldA}.
  - Let {typeB} be the type of {fieldB}.
  - {SameTypeShape(typeA, typeB)} must be true.

**Explanatory Text**

The input fields of input objects with the same name must be mergeable. This rule ensures that input objects with the same name in different source schemas have fields that can be merged consistently without conflicts.

Input fields are considered mergeable when they share the same name and have compatible types. The compatibility of types is determined by their structure (e.g., lists), excluding nullability. Mergeable input fields with different nullability are considered mergeable, and the resulting merged field will be the most permissive of the two.

In this example, the field `field` in `Input1` has compatible types across
source schemas, making them mergeable:

```graphql example
input Input1 {
  field: String!
}

input Input1 {
  field: String
}
```

Fields are also consodered mergable if they have different nullability defined across 
Here, the field `tags` in `Input1` is a list type with compatible inner types,
satisfying the mergeable criteria:

```graphql example
input Input1 {
  tags: [String!]
}

input Input1 {
  tags: [String]!
}

input Input1 {
  tags: [String]
}
```

In this counter-example, the field `field` in `Input1` has incompatible types
(`String` and `DateTime`), making them not mergeable:

```graphql counter-example
input Input1 {
  field: String!
}

input Input1 {
  field: DateTime!
}
```

Here, the field `tags` in `Input1` is a list type with incompatible inner types
(`String` and `DateTime`), violating the mergeable rule:

```graphql counter-example
input Input1 {
  tags: [String]
}

input Input1 {
  tags: [DateTime]
}
```

### Merge

### Post Merge Validation

## Validate Satisfiability
