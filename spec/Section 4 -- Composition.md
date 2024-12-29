# Schema Composition

The schema composition describes the process of merging multiple source schemas
into a single GraphQL schema, known as the _composite execution schema_, which
is a valid GraphQL schema annotated with execution directives. This composite
execution schema is the output of the schema composition process. The schema
composition process is divided into three main steps: **Validate Source
Schemas**, **Merge Source Schemas**, and **Validate Satisfiability**, which are
run in sequence to produce the composite execution schema.

## Validate Source Schemas

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

`OUTPUT_FIELD_TYPES_NOT_MERGEABLE`

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
  - {SameTypeShape(typeA, typeB)} must be true.

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
    - For each {argument} in {directive}:
      - {IsAccessible(argument)} must be true.

**Explanatory Text**

This rule ensures that certain essential elements of a GraphQL schema,
particularly built-in scalars, directive arguments, and introspection types,
cannot be marked as `@inaccessible`. These types are fundamental to GraphQL.
Making these elements inaccessible would break core GraphQL functionality.

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
# Source schema A
type Product {
  name(language: String = "en"): String
}

# Source schema B
type Product {
  name(language: String = "en") @external: String
}
```

Here, the `name` field on `Product` is defined in one source schema and marked
as `@external` in another. The argument `language` has different default values
in the two source schemas, violating the rule:

```graphql counter-example
# Source schema A
type Product {
  name(language: String = "en"): String
}

# Source schema B
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
# Source schema A
type Product {
  name(language: String = "en"): String
}

# Source schema B
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
# Source schema A
type Product {
  name(language: String): String
}

# Source schema B
type Product {
  name(language: String): String @external
}
```

Here, the `@external` field in source schema B is missing the `language`
argument that is present in the base field definition in source schema A,
violating the rule:

```graphql counter-example
# Source schema A
type Product {
  name(language: String): String
}

# Source schema B
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
# Source schema A
type Product {
  name(language: Language): String
}

# Source schema B
type Product {
  name(language: Language): String
}
```

In this example, the `@external` field's `language` argument type does not match
the base field's `language` argument type (`Language` vs. `String`), violating
the rule:

```graphql example
# Source schema A
type Product {
  name(language: Language): String
}

# Source schema B
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
# Source schema A
type Product {
  id: ID
  name: String
}

# Source schema B
type Product {
  id: ID
  name: String @external
}
```

In this example, the `name` field on `Product` is marked as `@external` in
source schema B but has no non-`@external` declaration in any other source
schema, violating the rule:

```graphql counter-example
# Source schema A
type Product {
  id: ID
}

# Source schema B
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
# Source schema A
type Product {
  name: String
}

# Source schema B
type Product {
  name: String @external
}
```

In this example, the `@external` field `name` has a return type of `ProductName`
that doesn't match the base field's return type `String`, violating the rule:

```graphql counter-example
# Source schema A
type Product {
  name: String
}

# Source schema B
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
# Source schema A
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
# Source schema A
type Product {
  title: String @external
  author: Author
}
```

### Root Mutation Used

**Error Code**

`ROOT_MUTATION_USED`

**Severity**

ERROR

**Formal Specification**

- Let {schemas} be the set of all source schemas.
- For each {schema} in {schemas}:
  - Let {rootMutation} be the root mutation type defined in {schema}, if it
    exists.
  - Let {namedMutationType} be the type with the name `Mutation` in {schema}, if
    it exists.
  - If {rootMutation} is defined:
    - {rootMutation} must be named `Mutation`.
  - Otherwise, {namedMutationType} must not be defined.

**Explanatory Text**

This rule enforces that, for any source schema, if a root mutation type is
defined, it must be named `Mutation`. Defining a root mutation type with a name
other than `Mutation` or using a differently named type alongside a type
explicitly named `Mutation` creates inconsistencies in schema design and
violates the composite schema specification.

**Examples**

Valid example:

```graphql example
schema {
  mutation: Mutation
}

type Mutation {
  createProduct(name: String): Product
}

type Product {
  id: ID!
  name: String
}
```

The following counter-example violates the rule because `RootMutation` is used
as the root mutation type, but a type named `Mutation` is also defined.

```graphql counter-example
schema {
  mutation: RootMutation
}

type RootMutation {
  createProduct(name: String): Product
}

type Mutation {
  deprecatedField: String
}
```

### Root Query Used

**Error Code**

`ROOT_QUERY_USED`

**Severity**

ERROR

**Formal Specification**

- Let {schemas} be the set of all source schemas.
- For each {schema} in {schemas}:
  - Let {rootQuery} be the root mutation type defined in {schema}, if it exists.
  - Let {namedQueryType} be the type with the name `Query` in {schema}, if it
    exists.
  - If {rootQuery} is defined:
    - {rootQuery} must be named `Query`.
  - Otherwise, {namedQueryType} must not be defined.

**Explanatory Text**

This rule enforces that the root query type in any source schema must be named
`Query`. Defining a root query type with a name other than `Query` or using a
differently named type alongside a type explicitly named `Query` creates
inconsistencies in schema design and violates the composite schema
specification.

**Examples**

Valid example:

```graphql example
schema {
  query: Query
}

type Query {
  product(id: ID!): Product
}

type Product {
  id: ID!
  name: String
}
```

The following counter-example violates the rule because `RootQuery` is used as
the root query type, but a type named `Query` is also defined.

```graphql counter-example
schema {
  query: RootQuery
}

type RootQuery {
  product(id: ID!): Product
}

type Query {
  deprecatedField: String
}
```

### Root Subscription Used

**Error Code**

`ROOT_SUBSCRIPTION_USED`

**Severity**

ERROR

**Formal Specification**

- Let {schemas} be the set of all source schemas.
- For each {schema} in {schemas}:
  - Let {rootSubscription} be the root mutation type defined in {schema}, if it
    exists.
  - Let {namedSubscriptionType} be the type with the name `Subscription` in
    {schema}, if it exists.
  - If {rootSubscription} is defined:
    - {rootSubscription} must be named `Subscription`.
  - Otherwise, {namedSubscriptionType} must not be defined.

**Explanatory Text**

This rule enforces that, for any source schema, if a root subscription type is
defined, it must be named `Subscription`. Defining a root subscription type with
a name other than `Subscription` or using a differently named type alongside a
type explicitly named `Subscription` creates inconsistencies in schema design
and violates the composite schema specification.

**Examples**

Valid example:

```graphql example
schema {
  subscription: Subscription
}

type Subscription {
  productCreated: Product
}

type Product {
  id: ID!
  name: String
}
```

The following counter-example violates the rule because `RootSubscription` is
used as the root subscription type, but a type named `Subscription` is also
defined.

```graphql counter-example
schema {
  subscription: RootSubscription
}

type RootSubscription {
  productCreated: Product
}

type Subscription {
  deprecatedField: String
}
```

### Key Fields Select Invalid Type

**Error Code**

`KEY_FIELDS_SELECT_INVALID_TYPE`

**Severity**

ERROR

**Formal Specification**

- Let {schema} be the set of all source schemas.
  - Let {types} be the set of all object or interface types that are annotated
    with the `@key` directive in {schema}.
  - For each {type} in {types}:
    - Let {keyDirectives} be the set of all `@key` directives on {type}.
    - For each {keyDirective} in {keyDirectives}
      - Let {keyFields} be the set of all fields (including nested) referenced
        by the `fields` argument of {keyDirective}.
      - For each {field} in {keyFields}:
        - Let {fieldType} be the type of {field}.
        - {fieldType} must not be a `List`, `Interface`, or `Union` type.

**Explanatory Text**

The `@key` directive is used to define the set of fields that uniquely identify
an entity. These fields must reference scalars or object types to ensure a valid
and consistent representation of the entity across schemas. Fields of types
`List`, `Interface`, or `Union` cannot be part of a `@key` because they do not
have a well-defined unique value.

**Examples**

In this valid example, the `Product` type has a valid `@key` directive
referencing the scalar field `sku`.

```graphql example
type Product @key(fields: "sku") {
  sku: String!
  name: String
}
```

In the following counter-example, the `Product` type has an invalid `@key`
directive referencing a field (`featuredItem`) whose type is an interface,
violating the rule.

```graphql counter-example
type Product @key(fields: "featuredItem { id }") {
  featuredItem: Node!
  sku: String!
}

interface Node {
  id: ID!
}
```

In this counter example, the `@key` directive references a field (`tags`) of
type `List`, which is also not allowed.

```graphql counter-example
type Product @key(fields: "tags") {
  tags: [String!]!
  sku: String!
}
```

In this counter example, the `@key` directive references a field
(`relatedItems`) of type `Union`, which violates the rule.

```graphql counter-example
type Product @key(fields: "relatedItems") {
  relatedItems: Related!
  sku: String!
}

union Related = Product | Service

type Service {
  id: ID!
}
```

### Key Directive in Fields Argument

**Error Code**

`KEY_DIRECTIVE_IN_FIELDS_ARG`

**Severity**

ERROR

**Formal Specification**

- Let {schema} be the set of all source schemas.
  - Let {types} be the set of all object and interface types in {schema}.
  - For each {type} in {types}:
    - Let {keyDirectives} be the set of all `@key` directives on {type}.
    - For each {keyDirective} in {keyDirectives}:
      - Let {fields} be the string value of the `fields` argument of
        {keyDirective}.
      - {fields} must not contain a directive application.

**Explanatory Text**

The `@key` directive specifies the set of fields used to uniquely identify an
entity. The `fields` argument must consist of a valid GraphQL selection set that
does not include any directive applications. Directives in the `fields` argument
are not supported.

**Examples**

In this example, the `fields` argument of the `@key` directive does not include
any directive applications, satisfying the rule.

```graphql example
type User @key(fields: "id name") {
  id: ID!
  name: String
}
```

In this counter-example, the `fields` argument of the `@key` directive includes
a directive application `@lowercase`, which is not allowed.

```graphql counter-example
directive @lowercase on FIELD_DEFINITION

type User @key(fields: "id name @lowercase") {
  id: ID!
  name: String
}
```

In this example, the `fields` argument includes a directive application
`@lowercase` nested inside the selection set, which is also invalid.

```graphql counter-example
directive @lowercase on FIELD_DEFINITION

type User @key(fields: "id name { firstName @lowercase }") {
  id: ID!
  name: FullName
}

type FullName {
  firstName: String
  lastName: String
}
```

### Key Fields Has Arguments

**Error Code**

`KEY_FIELDS_HAS_ARGS`

**Severity**

ERROR

**Formal Specification**

- Let {schema} be the set of all source schemas.
  - Let {types} be the set of all object types that are annotated with the
    `@key` directive in {schema}.
  - For each {type} in {types}:
    - Let {keyFields} be the set of fields referenced by the `fields` argument
      of the `@key` directive on {type}.
    - For each {field} in {keyFields}:
      - HasKeyFieldsArguments(field) must be true.

HasKeyFieldsArguments(field):

- If {field} has arguments:
  - return false
- If {field} has a selection set:
  - Let {subFields} be the set of all fields in the selection set of {field}.
  - For each {subField} in {subFields}:
    - HasKeyFieldsArguments(subField) must be true.
- return true

**Explanatory Text**

The `@key` directive is used to define the set of fields that uniquely identify
an entity. These fields must not include any field that is defined with
arguments, as arguments introduce variability that prevents consistent and valid
entity resolution across schemas. Fields included in the `fields` argument of
the `@key` directive must be static and consistently resolvable.

**Examples**

In this example, the `User` type has a valid `@key` directive that references
the argument-free fields `id` and `name`.

```graphql example
type User @key(fields: "id name") {
  id: ID!
  name: String
  tags: [String]
}
```

In this counter-example, the `@key` directive references a field (`tags`) that
is defined with arguments (`limit`), which is not allowed.

```graphql counter-example
type User @key(fields: "id tags") {
  id: ID!
  tags(limit: Int = 10): [String]
}
```

### Key Invalid Syntax

**Error Code**  
`KEY_INVALID_SYNTAX`

**Severity**  
ERROR

**Formal Specification**

- Let {schemas} be the set of all source schemas.
  - Let {types} be the set of all object or interface types in each {schema}.
  - For each {type} in {types}:
    - Let {keyDirectives} be the set of all `@key` directives on {type}.
    - For each {keyDirective} in {keyDirectives}:
      - Let {fieldsArg} be the string value of the `fields` argument of
        {keyDirective}.
      - Attempt to parse {fieldsArg} as a valid GraphQL selection set.
      - Parsing must **not** fail (e.g., missing braces, invalid tokens,
        unbalanced curly braces, or other syntax errors).

**Explanatory Text**

Each `@key` directive must specify the fields that uniquely identify an entity
using a valid GraphQL selection set in its `fields` argument. If the `fields`
argument string is syntactically incorrect—missing closing braces, containing
invalid tokens, or otherwise malformed - it cannot be composed into a valid
schema and triggers the `KEY_INVALID_SYNTAX` error.

**Examples**

In this valid scenario, the `fields` argument is a correctly formed selection
set: `"sku featuredItem { id }"` is properly balanced and contains no syntax
errors.

```graphql example
type Product @key(fields: "sku featuredItem { id }") {
  sku: String!
  featuredItem: Node!
}

interface Node {
  id: ID!
}
```

Here, the selection set `"featuredItem { id"` is missing the closing brace `}`.
It is thus invalid syntax, causing a `KEY_INVALID_SYNTAX` error.

```graphql counter-example
type Product @key(fields: "featuredItem { id") {
  featuredItem: Node!
  sku: String!
}

interface Node {
  id: ID!
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
