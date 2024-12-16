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

#### Enum Type Default Value Uses Inaccessible Value

**Error Code**

`ENUM_TYPE_DEFAULT_VALUE_INACCESSIBLE`

**Formal Specification**

- {ValidateArgumentDefaultValues()} must be true.
- {ValidateInputFieldDefaultValues()} must be true.

ValidateArgumentDefaultValues():

- Let {arguments} be all arguments of fields and directives across all source
  schemas
- For each {argument} in {arguments}
  - If {IsExposed(argument)} is true and has a default value:
    - Let {defaultValue} be the default value of {argument}
    - If not {ValidateDefaultValue(defaultValue)}
      - return false
- return true

ValidateInputFieldDefaultValues():

- Let {inputFields} be all input fields across all source schemas
- For each {inputField} in {inputFields}:
  - If {IsExposed(inputField)} is true and {inputField} has a default value:
    - Let {defaultValue} be the default value of {inputField}
    - If {ValidateDefaultValue(defaultValue)} is false
      - return false
- return true

ValidateDefaultValue(defaultValue):

- If {defaultValue} is a ListValue:
  - For each {valueNode} in {defaultValue}:
    - If {ValidateDefaultValue(valueNode)} is false
      - return false
- If {defaultValue} is an ObjectValue:
  - Let {objectFields} be a list of all fields of {defaultValue}
  - Let {fields} be a list of all fields {objectFields} are referring to
  - For each {field} in {fields}:
    - If {IsExposed(field)} is false
      - return false
  - For each {objectField} in {objectFields}:
    - Let {value} be the value of {objectField}
    - return {ValidateDefaultValue(value)}
- If {defaultValue} is an EnumValue:
  - If {IsExposed(defaultValue)} is false
    - return false
- return true

**Explanatory Text**

This rule ensures that inaccessible enum values are not exposed in the composed
schema through default values. Output field arguments, input fields, and
directive arguments must only use enum values as their default value when not
annotated with the `@inaccessible` directive.

In this example the `FOO` value in the `Enum1` enum is not marked with
`@inaccessible`, hence it does not violate the rule.

```graphql
type Query {
  field(type: Enum1 = FOO): [Baz!]!
}

enum Enum1 {
  FOO
  BAR
}
```

The following example violates this rule because the default value for the field
`field` in type `Input1` references an enum value (`FOO`) that is marked as
`@inaccessible`.

```graphql counter-example
type Query {
  field(arg: Enum1 = FOO): [Baz!]!
}

input Input1 {
  field: Enum1 = FOO
}

directive @directive1(arg: Enum1 = FOO) on FIELD_DEFINITION

enum Enum1 {
  FOO @inaccessible
  BAR
}
```

```graphql counter-example
type Query {
  field(arg: Input1 = { field2: "ERROR" }): [Baz!]!
}

directive @directive1(arg: Input1 = { field2: "ERROR" }) on FIELD_DEFINITION

input Input1 {
  field1: String
  field2: String @inaccessible
}
```

#### Output Field Types Mergeable

**Error Code**

OUTPUT_FIELD_TYPES_NOT_MERGEABLE

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
    - {FieldsAreMergeable(fields)} must be true.

FieldsAreMergeable(fields):

- Given each pair of members {fieldA} and {fieldB} in {fields}:
  - Let {typeA} be the type of {fieldA}
  - Let {typeB} be the type of {fieldB}
  - {SameOutputTypeShape(typeA, typeB)} must be true.

**Explanatory Text**

Fields on objects or interfaces that have the same name are considered
semantically equivalent and mergeable when they have a mergeable field type.

Fields with the same type are mergeable.

```graphql example
type User {
  birthdate: String
}

type User {
  birthdate: String
}
```

Fields with different nullability are mergeable, resulting in a merged field
with a nullable type.

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

#### Disallowed Inaccessible Elements

**Error Code**

`DISALLOWED_INACCESSIBLE`

**Severity**

ERROR

**Formal Specification**

- Let {type} be the set of all types from all source schemas.
- For each {type} in {types}:
  - If {type} is a built-in scalar type or introspection type:
    - {IsAccessible(type)} must be true.
    - For each {field} in {type}:
      - {IsAccessible(field)} must be true.
      - For each {argument} in {field}:
        - {IsAccessible(argument)} must be true.
- For each {directive} in {directives}:
  - If {directive} is a built-in directive:
    - {IsAccessible(directive)} must be true.
    - For each {argument} in {directive}:
      - {IsAccessible(argument)} must be true.

**Explanatory Text**

This rule ensures that certain essential elements of a GraphQL schema,
particularly built-in scalars, directives and introspection types, cannot be
marked as `@inaccessible`. These types are fundamental to GraphQL's. Making
these elements inaccessible would break core GraphQL functionalities.

Here, the `String` type is not marked as `@inaccessible`, which adheres to the
rule:

```graphql example
type Product {
  price: Float
  name: String
}
```

In this example, the `String` scalar is marked as `@inaccessible`. This violates
the rule because `String` is a required built-in type that cannot be
inaccessible:

```graphql counter-example
scalar String @inaccessible

type Product {
  price: Float
  name: String
}
```

In this example, the introspection type `__Type` is marked as `@inaccessible`.
This violates the rule because introspection types must remain accessible for
GraphQL introspection queries to work.

```graphql counter-example
type __Type @inaccessible {
  kind: __TypeKind!
  name: String
  fields(includeDeprecated: Boolean = false): [__Field!]
}
```

#### External Argument Default Mismatch

**Error Code**

`EXTERNAL_ARGUMENT_DEFAULT_MISMATCH`

**Severity**

ERROR

**Formal Specification**

- Let {typeNames} be the set of all output type names from all source schemas.
- For each {typeName} in {typeNames}
  - Let {types} be the set of all types with the name {typeName} from all source
    schemas.
  - Let {fieldNames} be the set of all field names from all types in {types}.
  - For each {fieldName} in {fieldNames}
    - Let {fields} be the set of all fields with the name {fieldName} from all
      types in {types}.
    - Let {externalFields} be the set of all fields in {fields} that are marked
      with `@external`.
    - If {externalFields} is not empty
      - Let {argumentNames} be the set of all argument names from all fields in
        {fields}.
      - For each {argumentName} in {argumentNames}
        - Let {arguments} be the set of all arguments with the name
          {argumentName} from all fields in {fields}.
        - Let {defaultValue} be the first default value found in {arguments}.
        - Let {externalArguments} be the set of all arguments with the name
          {argumentName} from all fields in {externalFields}.
        - For each {externalArgument} in {externalArguments}
          - The default value of {externalArgument} must be equal to
            {defaultValue}.

**Explanatory Text**

This rule ensures that arguments on fields marked as `@external` have default
values compatible with the corresponding arguments on fields from other source
schemas where the field is defined (non-`@external`). Since `@external` fields
represent fields that are resolved by other source schemas, their arguments and
defaults must match to maintain consistent behavior across different source
schemas.

Here, the `name` field on `Product` is defined in one source schema and marked
as `@external` in another. The argument `language` has the same default value in
both source schemas, satisfying the rule:

```graphql example
# Subgraph A
type Product {
  name(language: String = "en"): String
}

# Subgraph B
type Product {
  name(language: String = "en") @external: String
}
```

Here, the `name` field on `Product` is defined in one source schema and marked
as `@external` in another. The argument `language` has different default values
in the two source schemas, violating the rule:

```graphql counter-example
# Subgraph A
type Product {
  name(language: String = "en"): String
}

# Subgraph B
type Product {
  name(language: String = "de") @external: String
}
```

In the following counter example, the `name` field on `Product` is defined in
one source schema and marked as `@external` in another. The argument `language`
has a default value in the source schema where the field is defined, but it does
not have a default value in the source schema where the field is marked as
`@external`, violating the rule:

```graphql counter-example
# Subgraph A
type Product {
  name(language: String = "en"): String
}

# Subgraph B
type Product {
  name(language: String): String @external
}
```

#### External Argument Missing

**Error Code**

`EXTERNAL_ARGUMENT_MISSING`

**Severity**

ERROR

**Formal Specification**

- Let {typeNames} be the set of all output type names from all source schemas.
- For each {typeName} in {typeNames}
  - Let {types} be the set of all types with the name {typeName} from all source
    schemas.
  - Let {fieldNames} be the set of all field names from all types in {types}.
  - For each {fieldName} in {fieldNames}
    - Let {fields} be the set of all fields with the name {fieldName} from all
      types in {types}.
    - Let {externalFields} be the set of all fields in {fields} that are marked
      with `@external`.
    - Let {nonExternalFields} be the set of all fields in {fields} that are not
      marked with `@external`.
    - If {externalFields} is not empty
      - Let {argumentNames} be the set of all argument names from all fields in
        {nonExternalFields}
      - For each {argumentName} in {argumentNames}:
        - For each {externalField} in {externalFields}
          - {argumentName} must be present in the arguments of {externalField}.

**Explanatory Text**

This rule ensures that fields marked with `@external` have all the necessary
arguments that exist on the corresponding field definitions in other source
schemas. Each argument defined on the base field (the field definition in the
source source schema) must be present on the `@external` field in other source
schemas. If an argument is missing on an `@external` field, the field cannot be
resolved correctly, which is an inconsistency.

In this example, the `language` argument is present on both the `@external`
field in source schema B and the base field in source schema A, satisfying the
rule:

```graphql example
# Subgraph A
type Product {
  name(language: String): String
}

# Subgraph B
type Product {
  name(language: String): String @external
}
```

Here, the `@external` field in source schema B is missing the `language`
argument that is present in the base field definition in source schema A,
violating the rule:

```graphql counter-example
# Subgraph A
type Product {
  name(language: String): String
}

# Subgraph B
type Product {
  name: String @external
}
```

#### External Argument Type Mismatch

**Error Code**

`EXTERNAL_ARGUMENT_TYPE_MISMATCH`

**Severity**

ERROR

**Formal Specification**

- Let {typeNames} be the set of all output type names from all source schemas.
- For each {typeName} in {typeNames}
  - Let {types} be the set of all types with the name {typeName} from all source
    schemas.
  - Let {fieldNames} be the set of all field names from all types in {types}.
  - For each {fieldName} in {fieldNames}
    - Let {fields} be the set of all fields with the name {fieldName} from all
      types in {types}.
    - Let {externalFields} be the set of all fields in {fields} that are marked
      with `@external`.
    - Let {nonExternalFields} be the set of all fields in {fields} that are not
      marked with `@external`.
    - If {externalFields} is not empty
      - Let {argumentNames} be the set of all argument names from all fields in
        {nonExternalFields}
      - For each {argumentName} in {argumentNames}:
        - For each {externalField} in {externalFields}
          - Let {externalArgument} be the argument with the name {argumentName}
            from {externalField}.
          - {externalArgument} must strictly equal all arguments with the name
            {argumentName} from {nonExternalFields}.

**Explanatory Text**

This rule ensures that arguments on fields marked as `@external` have types
compatible with the corresponding arguments on the fields defined in other
source schemas. The arguments must have the exact same type signature, including
nullability and list nesting.

Here, the `@external` field's `language` argument has the same type (`Language`)
as the base field, satisfying the rule:

```graphql example
# Subgraph A
type Product {
  name(language: Language): String
}

# Subgraph B
type Product {
  name(language: Language): String
}
```

In this example, the `@external` field's `language` argument type does not match
the base field's `language` argument type (`Language` vs. `String`), violating
the rule:

```graphql example
# Subgraph A
type Product {
  name(language: Language): String
}

# Subgraph B
type Product {
  name(language: String): String
}
```

#### External Missing on Base

**Error Code**

`EXTERNAL_MISSING_ON_BASE`

**Severity**

ERROR

**Formal Specification**

- Let {typeNames} be the set of all output type names from all source schemas.
- For each {typeName} in {typeNames}
  - Let {types} be the set of all types with the name {typeName} from all source
    schemas.
  - Let {fieldNames} be the set of all field names from all types in {types}.
  - For each {fieldName} in {fieldNames}
    - Let {fields} be the set of all fields with the name {fieldName} from all
      types in {types}.
    - Let {externalFields} be the set of all fields in {fields} that are marked
      with `@external`.
    - Let {nonExternalFields} be the set of all fields in {fields} that are not
      marked with `@external`.
    - If {externalFields} is not empty
      - {nonExternalFields} must not be empty.

**Explanatory Text**

This rule ensures that any field marked as `@external` in a source schema is
actually defined (non-`@external`) in at least one other source schema. The
`@external` directive is used to indicate that the field is not usually resolved
by the source schema it is declared in, implying it should be resolvable by at
least one other source schema.

Here, the `name` field on `Product` is defined in source schema A and marked as
`@external` in source schema B, which is valid because there is a base
definition in source schema A:

```graphql example
# Subgraph A
type Product {
  id: ID
  name: String
}

# Subgraph B
type Product {
  id: ID
  name: String @external
}
```

In this example, the `name` field on `Product` is marked as `@external` in
source schema B but has no non-`@external` declaration in any other source
schema, violating the rule:

```graphql counter-example
# Subgraph A
type Product {
  id: ID
}

# Subgraph B
type Product {
  id: ID
  name: String @external
}
```

#### External Type Mismatch

**Error Code**

`EXTERNAL_TYPE_MISMATCH`

**Severity**

ERROR

**Formal Specification**

- Let {typeNames} be the set of all output type names from all source schemas.
- For each {typeName} in {typeNames}
  - Let {types} be the set of all types with the name {typeName} from all source
    schemas.
  - Let {fieldNames} be the set of all field names from all types in {types}.
  - For each {fieldName} in {fieldNames}
    - Let {fields} be the set of all fields with the name {fieldName} from all
      types in {types}.
    - Let {externalFields} be the set of all fields in {fields} that are marked
      with `@external`.
    - Let {nonExternalFields} be the set of all fields in {fields} that are not
      marked with `@external`.
    - For each {externalField} in {externalFields}
      - The type of {externalField} must strictly equal all types of
        {nonExternalFields}.

**Explanatory Text**

This rule ensures that a field marked as `@external` has a return type
compatible with the corresponding field defined in other source schemas. Fields
with the same name must represent the same data type to maintain schema
consistency

Here, the `@external` field `name` has the same return type (`String`) as the
base field definition, satisfying the rule:

```graphql example
# Subgraph A
type Product {
  name: String
}

# Subgraph B
type Product {
  name: String @external
}
```

In this example, the `@external` field `name` has a return type of `ProductName`
that doesn't match the base field's return type `String`, violating the rule:

```graphql counter-example
# Subgraph A
type Product {
  name: String
}

# Subgraph B
type Product {
  name: ProductName @external
}
```

#### External Unused

**Error Code**

`EXTERNAL_UNUSED`

**Severity**

ERROR

**Formal Specification**

- For each {schema} in all source schemas
  - Let {types} be the set of all composite types (object, interface) in
    {schema}.
  - For each {type} in {types}:
    - Let {fields} be the set of fields for {type}.
    - For each {field} in {fields}:
      - If {field} is marked with `@external`:
        - Let {referencingFields} be the set of fields in {schema} that
          reference {type}.
        - {referencingFields} must contain at least one field that references
          {field} in `@provides`

**Explanatory Text**

This rule ensures that every field marked as `@external` in a source schema is
actually used by that source schema in a `@provides` directive.

**Examples**

In this example, the `name` field is marked with `@external` and is used by the
`@provides` directive, satisfying the rule:

```graphql example
# Subgraph A
type Product {
  id: ID
  name: String @external
}

type Query {
  productByName(name: String): Product @provides(fields: "name")
}
```

In this example, the `name` field is marked with `@external` but is not used by
the `@provides` directive, violating the rule:

```graphql counter-example
# Subgraph A
type Product {
  title: String @external
  author: Author
}
```

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
  - If {IsAccessible(field)} is true
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
