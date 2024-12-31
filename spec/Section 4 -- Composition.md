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

#### **Input With Missing Required Fields**

**Error Code:** 

REQUIRED_INPUT_FIELD_MISSING_IN_SOME_SUBGRAPH

**Severity:** 

ERROR

**Formal Specification:**

- Let {typeNames} be the set of all input object types from all source schemas that are not declared as `@inaccessible`.
- For each {typeName} in {typeNames}:
  - Let `{types}` be the list of all input object types from different source schemas with the name {typeName}.
  - `{AreTypesConsistent(types)}` must be true.

**Function Definition:**

`AreTypesConsistent(inputs):`

- Let `{requiredFields}` be the intersection of all required field names across all input objects in `{inputs}`.
- For each `{input}` in `{inputs}`:
  - Let `{inputFields}` be the set of all field names of required fields in `{input}`.
  - `{inputFields}` must equal `{requiredFields}`.

**Explanatory Text**

Input types are merged by intersection, meaning that the merged input type will have all fields that are present in all input types with the same name.
This rule ensures that input object types with the same name across different subgraphs share a consistent set of required fields. 

When an input object is defined across multiple subgraphs, this rule ensures that any required field present in one subgraph is present in all others defining the input object. 
This allows for reliable usage of input objects in queries across subgraphs.

### Examples

**Valid Example:**

If all subgraphs define `Input1` with the required field `field1`, the rule is satisfied:

```graphql
# Subgraph 1
input Input1 {
  field1: String!
  field2: String
}

# Subgraph 2
input Input1 {
  field1: String!
  field3: Int
}
```

**Invalid Example:**

If `field1` is required in one subgraph but missing in another, this violates the rule:

```graphql
# Subgraph 1
input Input1 {
  field1: String!
  field2: String
}

# Subgraph 2
input Input1 {
  field2: String
  field3: Int
}
```

In this invalid case, `field1` is mandatory in `Subgraph 1` but not defined in `Subgraph 2`, causing inconsistency in required fields across subgraphs.

### Merge

### Post Merge Validation

## Validate Satisfiability
