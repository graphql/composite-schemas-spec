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

#### Input Field Default Mismatch

**Error Code**

`INPUT_FIELD_DEFAULT_MISMATCH`

**Formal Specification**

- Let {inputFieldsByName} be a map where the key is the name of an input field
  and the value is a list of input fields from different source schemas from the same
  type with the same name.
- For each {inputFields} in {inputFieldsByName}:
  - Let {defaultValues} be a set containing the default values of each input
    field in {inputFields}.
  - If the size of {defaultValues} is greater than
    - {InputFieldsHaveConsistentDefaults(inputFields)} must be false.

InputFieldsHaveConsistentDefaults(inputFields):

- Given each pair of input fields {inputFieldA} and {inputFieldB} in
  {inputFields}:
  - If the default value of {inputFieldA} is not equal to the default value of
    {inputFieldB}:
    - return false
- return true

**Explanatory Text**

Input fields in different source schemas that have the same name are required to have
consistent default values. This ensures that there is no ambiguity or
inconsistency when merging input fields from different source schemas.

A mismatch in default values for input fields with the same name across
different source schemas will result in a schema composition error.

In the the following example both source schemas have an input field `field1` with
the same default value. This is valid:

```graphql example
input Filter {
  field1: Enum1 = VALUE1
}

enum Enum1 {
  VALUE1
  VALUE2
}

input Filter {
  field1: Enum1 = VALUE1
}

enum Enum1 {
  VALUE1
  VALUE2
}
```

In the following example both source schemas define an input field `field1` with
different default values. This is invalid:

```graphql counter-example
input Filter {
  field1: Int = 10
}

input Filter {
  field1: Int = 20
}
```

### Merge

### Post Merge Validation

## Validate Satisfiability
