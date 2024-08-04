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

#### Input With Different Fields

**Error Code**

INPUT_OBJECT_FIELDS_DIFFER

**Formal Specification**

- Let {inputsByName} be a map where the key is the name of an input object type, and the value is a list of all input object types from different source schemas with that name.
- For each {listOfInputs} in {inputsByName}:
  - {InputFieldsAreMergeable(listOfInputs)} must be true.

InputFieldsAreMergeable(inputs):

- Let {fields} be the set of all field names of the first input object in {inputs}.
- For each {input} in {inputs}:
  - Let {inputFields} be the set of all field names of {input}.
  - {fields} must be equal to {inputFields}.

**Explanatory Text**

This rule ensures that input object types with the same name across different source schemas have identical sets of field names. 
Consistency in input object fields across source schemas is required to avoid conflicts and ambiguities in the composed schema.
This rule only checks that the field names are the same, not that the field types are the same. 
Field types are checked by the [Input Field Types mergeable](#sec-Input-Field-Types-mergeable) rule.

When an input object is defined with differing fields across source schemas, it can lead to issues in query execution. 
A field expected in one subgraph might be absent in another, leading to undefined behavior. 
This rule prevents such inconsistencies by enforcing that all instances of the same named input object across subgraphs have a matching set of field names.

In this example, both subgraphs define `Input1` with the same field `field1`, satisfying the rule:

```graphql example
input Input1 {
  field1: String
}

input Input1 {
  field1: String
}
```

Here, the two definitions of `Input1` have different fields (`field1` and `field2`), violating the rule:

```graphql counter-example
input Input1 {
  field1: String
}

input Input1 {
  field2: String
}
```


### Merge

### Post Merge Validation

## Validate Satisfiability
