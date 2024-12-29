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

### Key Invalid Fields

**Error Code**

`KEY_INVALID_FIELDS`

**Severity**

ERROR

**Formal Specification**

- Let {schema} be the set of all source schemas.
  - Let {types} be the set of all object and interface types in {schema}.
  - For each {type} in {types}:
    - Let {keyDirectives} be the set of all `@key` directives on {type}.
    - For each {keyDirective} in {keyDirectives}:
      - Let {fieldsArg} be the string value of the `fields` argument of
        {keyDirective}.
      - Let {selections} be the set of fields in the selection set of
        {fieldsArg}.
      - For each {selection} in {selections}:
        - {IsValidKeyField(selection, type)} must be true.

IsValidKeyField(selection, type):

- If {selection} is not defined on {type}:
  - return false
- If {selection} has a selection set:
  - Let {subType} be the return type of {field}.
  - Let {subFields} be the set of all fields in the selection set of {field}.
  - For each {subField} in {subFields}:
    - {IsValidKeyField(subField, subType)} must be true.
- return true

**Explanatory Text**

Even if the selection set for `@key(fields: "…")` is syntactically valid, field
reference within that selection set must also refer to **actual** fields on the
annotated type. This includes nested selections, which must appear on the
corresponding return type. If any referenced field is missing or incorrectly
named, composition fails with a `KEY_INVALID_FIELDS` error because the entity
key cannot be resolved correctly.

**Examples**

In this valid example, the `fields` argument of the `@key` directive is properly
defined with valid syntax and references existing fields.

```graphql example
type Product @key(fields: "sku featuredItem { id }") {
  sku: String!
  featuredItem: Node!
}

interface Node {
  id: ID!
}
```

In this counter-example, the `fields` argument of the `@key` directive
references a field `id`, which does not exist on the `Product` type.

```graphql counter-example
type Product @key(fields: "id") {
  sku: String!
}
```

### Provides Directive in Fields Argument

**Error Code**

`PROVIDES_DIRECTIVE_IN_FIELDS_ARG`

**Severity**

ERROR

**Formal Specification**

- Let {schema} be the set of all source schemas.
  - Let {fieldsWithProvides} be the set of all fields annotated with the
    `@provides` directive in {schema}.
  - For each {field} in {fieldsWithProvides}:
    - Let {fields} be the selected fields of the `fields` argument of the
      `@provides` directive on {field}.
    - For each {selection} in {fields}:
      - {HasProvidesDirective(selection)} must be false

HasProvidesDirective(selection):

- If {selection} has a directive application:
  - return true
- If {selection} has a selection set:
  - Let {subSelections} be the selections in {selection}
  - For each {subSelection} in {subSelections}:
    - If {HasProvidesDirective(subSelection)} is true
      - return true

**Explanatory Text**

The `@provides` directive is used to specify the set of fields on an object type
that a resolver provides for the parent type. The `fields` argument must consist
of a valid GraphQL selection set without any directive applications, as
directives within the `fields` argument are not supported.

**Examples**

In this example, the `fields` argument of the `@provides` directive does not
have any directive applications, satisfying the rule.

```graphql example
type User @key(fields: "id name") {
  id: ID!
  name: String
  profile: Profile @provides(fields: "name")
}

type Profile {
  id: ID!
  name: String
}
```

In this counter-example, the `fields` argument of the `@provides` directive has
a directive application `@lowercase`, which is not allowed.

```graphql counter-example
directive @lowercase on FIELD_DEFINITION

type User @key(fields: "id name") {
  id: ID!
  name: String
  profile: Profile @provides(fields: "name @lowercase")
}

type Profile {
  id: ID!
  name: String
}
```

### Provides Fields Has Arguments

**Error Code**

`PROVIDES_FIELDS_HAS_ARGS`

**Severity**

ERROR

**Formal Specification**

- Let {schema} be the set of all source schemas.
  - Let {fieldsWithProvides} be the set of all fields annotated with the
    `@provides` directive in {schema}.
  - For each {field} in {fieldsWithProvides}:
    - Let {selections} be the field selections of the `fields` argument of the
      `@provides` directive on {field}.
    - Let {type} be the return type of {field}
    - For each {selection} in {selections}:
      - {ProvidesHasArguments(selection, type)} must be false

ProvidesHasArguments(selection, type):

- Let {field} be the field of {type} selected by {selection}
- If {field} has arguments:
  - return true
- If {selection} has a selection set:
  - Let {subSelections} be the selections in {selection}
  - Let {subType} be the return type of {field}
  - For each {subSelection} in {subSelections}:
    - If {ProvidesHasArguments(subField, subSelection)} is true
      - return true

**Explanatory Text**

The `@provides` directive specifies fields that a resolver provides for the
parent type. The `fields` argument must reference fields that do not have
arguments, as fields with arguments introduce variability that is incompatible
with the consistent behavior expected of `@provides`.

**Examples**

```graphql example
type User @key(fields: "id") {
  id: ID!
  tags: [String]
}

type Article @key(fields: "id") {
  id: ID!
  author: User! @provides(fields: "tags")
}
```

This violates the rule because the `tags` field referenced in the `fields`
argument of the `@provides` directive is defined with arguments
(`limit: UserType = ADMIN`).

```graphql counter-example
type User @key(fields: "id") {
  id: ID!
  tags(limit: UserType = ADMIN): [String]
}

enum UserType {
  REGULAR
  ADMIN
}

type Article @key(fields: "id") {
  id: ID!
  author: User! @provides(fields: "tags")
}
```

### Provides Fields Missing External

**Error Code**

`PROVIDES_FIELDS_MISSING_EXTERNAL`

**Severity**

ERROR

**Formal Specification**

- Let {schemas} be the set of all source schemas.
- For each {schema} in {schemas}
  - Let {objectTypes} be the set of all object types in {schema}.
  - For each {objectType} in {objectTypes}:
    - Let {providingFields} be the set of fields on {objectType} annotated with
      `@provides`.
    - For each {field} in {providingFields}:
      - Let {referencedFields} be the set of fields referenced by the `fields`
        argument of the `@provides` directive on {field}.
      - For each {referencedField} in {referencedFields}:
        - If {referencedField} is **not** marked as `@external`
          - Produce a `PROVIDES_FIELDS_MISSING_EXTERNAL` error.

**Explanatory Text**

The `@provides` directive indicates that an object type field will supply
additional fields belonging to the return type in this execution specific path.
Any field listed in the `@provides(fields: ...)` argument must therefore be
_external_ in the local schema, meaning that the local schema itself does
**not** provide it.

This rule disallows selecting non-external fields in a `@provides` selection
set. If a field is already provided by the same schema in all execution paths,
there is no need to `@provide`.

**Examples**

Here, the `Order` type from this schema is providing fields on `User` through
`@provides`. The `name` field of `User` is **not** defined in this schema; it is
declared with `@external` indicating that the `name` field comes from elsewhere.
Thus, referencing `name` under `@provides(fields: "name")` is valid.

```graphql example
type Order {
  id: ID!
  customer: User @provides(fields: "name")
}

type User @key(fields: "id") {
  id: ID!
  name: String @external
}
```

In this counter-example, `User.address` is **not** marked as `@external` in the
same schema that applies `@provides`. This means the schema already provides the
`address` field in all possible paths, so using `@provides(fields: "address")`
is invalid.

```graphql counter-example
type User {
  id: ID!
  address: String
}

type Order {
  id: ID!
  buyer: User @provides(fields: "address")
}
```

### Query Root Type Inaccessible

**Error Code**

`QUERY_ROOT_TYPE_INACCESSIBLE`

**Severity**

ERROR

**Formal Specification**

- Let {schemas} be the set of all source schemas.
- For each {schema} in {schemas}:
  - Let {queryType} be the query operation type defined in {schema}.
  - If {queryType} is annotated with `@inaccessible`:
    - Produce a `QUERY_ROOT_TYPE_INACCESSIBLE` error.

**Explanatory Text**

Every source schema that contributes to the final composite schema must expose a
public (accessible) root query type. Marking the root query type as
`@inaccessible` makes it invisible to the gateway, defeating its purpose as the
primary entry point for queries and lookups.

**Examples**

In this example, no `@inaccessible` annotation is applied to the query root, so
the rule is satisfied.

```graphql example
extend schema {
  query: Query
}

type Query {
  allBooks: [Book]
}

type Book {
  id: ID!
  title: String
}
```

Since the schema marks the query root type as `@inaccessible`, the rule is
violated. `QUERY_ROOT_TYPE_INACCESSIBLE` is raised because a schema’s root query
type cannot be hidden from consumers.

```graphql counter-example
extend schema {
  query: Query
}

type Query @inaccessible {
  allBooks: [Book]
}

type Book {
  id: ID!
  title: String
}
```

### Require Directive in Fields Argument

**Error Code**

`REQUIRE_DIRECTIVE_IN_FIELDS_ARG`

**Severity**

ERROR

**Formal Specification**

- Let {schemas} be the set of all source schemas.
- For each {schema} in {schemas}:
  - Let {compositeTypes} be the set of all composite types in {schema}.
  - For each {composite} in {compositeTypes}:
    - Let {fields} be the set of fields on {composite}
    - Let {arguments} be the set of all arguments on {fields}
    - For each {argument} in {arguments}:
      - If {argument} is **not** marked with `@require`:
        - Continue
      - Let {fieldsArg} be the value of the `fields` argument of the `@require`
        directive on {argument}.
      - If {fieldsArg} contains a directive application:
        - Produce a `REQUIRE_DIRECTIVE_IN_FIELDS_ARG` error.

**Explanatory Text**

The `@require` directive is used to specify fields on the same type that an
argument depends on in order to resolve the annotated field.  
When using `@require(fields: "…")`, the `fields` argument must be a valid
selection set string **without** any additional directive applications.  
Applying a directive (e.g., `@lowercase`) inside this selection set is not
supported and triggers the `REQUIRE_DIRECTIVE_IN_FIELDS_ARG` error.

**Examples**

In this valid usage, the `@require` directive’s `fields` argument references
`name` without any directive applications, avoiding the error.

```graphql example
type User @key(fields: "id name") {
  id: ID!
  profile(name: String! @require(fields: "name")): Profile
}

type Profile {
  id: ID!
  name: String
}
```

Because the `@require` selection (`name @lowercase`) includes a directive
application (`@lowercase`), this violates the rule and triggers a
`REQUIRE_DIRECTIVE_IN_FIELDS_ARG` error.

```graphql counter-example
type User @key(fields: "id name") {
  id: ID!
  name: String
  profile(name: String! @require(fields: "name @lowercase")): Profile
}

type Profile {
  id: ID!
  name: String
}
```

### Require Invalid Fields Type

**Error Code**

`REQUIRE_INVALID_FIELDS_TYPE`

**Severity**

ERROR

**Formal Specification**

- Let {schemas} be the set of all source schemas.
- For each {schema} in {schemas}:
  - Let {compositeTypes} be the set of all composite types in {schema}.
  - For each {composite} in {compositeTypes}:
    - Let {fields} be the set of fields on {composite}.
    - Let {arguments} be the set of all arguments on {fields}.
    - For each {argument} in {arguments}:
      - If {argument} is **not** annotated with `@require`:
        - Continue
      - Let {fieldsArg} be the value of the `fields` argument of the `@require`
        directive on {argument}.
      - If {fieldsArg} is **not** a string:
        - Produce a `REQUIRE_INVALID_FIELDS_TYPE` error.

**Explanatory Text**

When using the `@require` directive, the `fields` argument must always be a
string that defines a (potentially nested) selection set of fields from the same
type. If the `fields` argument is provided as a type other than a string (such
as an integer, boolean, or enum), the directive usage is invalid and will cause
schema composition to fail.

**Examples**

In the following example, the `@require` directive’s `fields` argument is a
valid string and satisfies the rule.

```graphql example
type User @key(fields: "id") {
  id: ID!
  profile(name: String! @require(fields: "name")): Profile
}

type Profile {
  id: ID!
  name: String
}
```

Since `fields` is set to `123` (an integer) instead of a string, this violates
the rule and triggers a `REQUIRE_INVALID_FIELDS_TYPE` error.

```graphql counter-example
type User @key(fields: "id") {
  id: ID!
  profile(name: String! @require(fields: 123)): Profile
}

type Profile {
  id: ID!
  name: String
}
```

### Require Invalid Syntax

**Error Code**

`REQUIRE_INVALID_SYNTAX`

**Severity**

ERROR

**Formal Specification**

- Let {schemas} be the set of all source schemas.
- For each {schema} in {schemas}
  - Let {compositeTypes} be the set of all composite types in {schema}.
  - For each {composite} in {compositeTypes}:
    - Let {fields} be the set of fields on {composite}.
    - Let {arguments} be the set of all arguments on {fields}.
    - For each {argument} in {arguments}:
      - If {argument} is **not** annotated with `@require`:
        - Continue
      - Let {fieldsArg} be the string value of the `fields` argument of the
        `@require` directive on {argument}.
      - {fieldsArg} must be be parsable as a valid selection map

**Explanatory Text**

The `@require` directive’s `fields` argument must be syntactically valid
GraphQL. If the selection map string is malformed (e.g., missing closing braces,
unbalanced quotes, invalid tokens), then the schema cannot be composed
correctly. In such cases, the error `REQUIRE_INVALID_SYNTAX` is raised.

**Examples**

In the following example, the `@require` directive’s `fields` argument is a
valid selection map and satisfies the rule.

```graphql example
type User @key(fields: "id") {
  id: ID!
  profile(name: String! @require(fields: "name")): Profile
}

type Profile {
  id: ID!
  name: String
}
```

In the following counter-example, the `@require` directive’s `fields` argument
has invalid syntax because it is missing a closing brace.

This violates the rule and triggers a `REQUIRE_INVALID_FIELDS` error.

```graphql counter-example
type Book {
  id: ID!
  title(lang: String! @require(fields: "author { name ")): String
}

type Author {
  name: String
}
```

### Type Definition Invalid

**Error Code**

`TYPE_DEFINITION_INVALID`

**Severity**

ERROR

**Formal Specification**

- Let {schema} be one of the source schemas.
- Let {types} be the set of built-in types (for example, `FieldSelectionMap`)
  defined by the composition specification.
- For each {type} in {types}:
  - {type} must strictly equal the built-in type defined by the composition
    specification.

**Explanatory Text**

Certain types are reserved in composite schema specification for specific
purposes and must adhere to the specification’s definitions. For example,
`FieldSelectionMap` is a built-in scalar that represents a selection of fields
as a string. Redefining these built-in types with a different kind (e.g., an
input object, enum, union, or object type) is disallowed and makes the
composition invalid.

This rule ensures that built-in types maintain their expected shapes and
semantics so the composed schema can correctly interpret them.

**Examples**

In the following counter-example, `FieldSelectionMap` is declared as an `input`
type instead of the required `scalar`. This leads to a `TYPE_DEFINITION_INVALID`
error because the defined scalar `FieldSelectionMap` is being overridden by an
incompatible definition.

```graphql counter-example
directive @require(field: FieldSelectionMap!) on ARGUMENT_DEFINITION

input FieldSelectionMap {
  fields: [String!]!
}
```

### Type Kind Mismatch

**Error Code**  
`TYPE_KIND_MISMATCH`

**Severity**  
ERROR

**Formal Specification**

- Let {schemas} be the set of all source schemas.
- For each type name {typeName} defined in at least one of these schemas:
  - Let {types} be the set of all types named {typeName} across all source
    schemas.
  - Let {typeKinds} be the set of
    [type kinds](https://spec.graphql.org/October2021/#sec-Type-Kinds) in
    {types}
  - {typeKinds} must contain exactly one element.

**Explanatory Text**

Each named type must represent the **same** kind of GraphQL type across all
source schemas. For instance, a type named `User` must consistently be an object
type, or consistently be an interface, and so forth. If one schema defines
`User` as an object type, while another schema declares `User` as an interface
(or input object, union, etc.), the schema composition process cannot merge
these definitions coherently.

This rule ensures semantic consistency: a single type name cannot serve
multiple, incompatible purposes in the final composed schema.

**Examples**

All schemas agree that `User` is an object type:

```graphql
# Schema A
type User {
  id: ID!
  name: String
}

# Schema B
type User {
  id: ID!
  email: String
}

# Schema C
type User {
  id: ID!
  joinedAt: String
}
```

In the following counter-example, `User` is defined as an object type in one of
the schemas and as an interface in another. This violates the rule and results

```graphql
# Schema A: `User` is an object type
type User {
  id: ID!
  name: String
}

# Schema B: `User` is an interface
extend interface User {
  id: ID!
  friends: [User!]!
}

# Schema C: `User` is an input object
extend input User {
  id: ID!
}
```

### Provides Invalid Syntax

**Error Code**  
`PROVIDES_INVALID_SYNTAX`

**Severity**  
ERROR

**Formal Specification**

- Let {schemas} be the set of all source schemas.
- For each {schema} in {schemas}
  - Let {fieldsWithProvides} be the set of all fields annotated with the
    `@provides` directive in {schema}.
  - For each {field} in {fieldsWithProvides}:
    - Let {fieldsArg} be the string value of the `fields` argument of the
      `@provides` directive on {field}.
    - {fieldsArg} must be a valid selection set string

**Explanatory Text**

The `@provides` directive’s `fields` argument must be a syntactically valid
selection set string, as if you were selecting fields in a GraphQL query. If the
selection set is malformed (e.g., missing braces, unbalanced quotes, or invalid
tokens), the schema composition fails with a `PROVIDES_INVALID_SYNTAX` error.

**Examples**

Here, the `@provides` directive’s `fields` argument is a valid selection set:

```graphql example
type User @key(fields: "id") {
  id: ID!
  address: Address @provides(fields: "street city")
}

type Address {
  street: String
  city: String
}
```

In this counter-example, the `fields` argument is missing a closing brace. It
cannot be parsed as a valid GraphQL selection set, triggering a
`PROVIDES_INVALID_SYNTAX` error.

```graphql counter-example
type User @key(fields: "id") {
  id: ID!
  address: Address @provides(fields: "{ street city ")
}
```

### Invalid GraphQL

**Error Code**  
`INVALID_GRAPHQL`

**Severity**  
ERROR

**Formal Specification**

- Let {schemas} be the set of all source schemas to be composed.
- For Each {schema} in {schemas}
  - {schema} must be a syntactically valid
  - {schama} must be a semantically valid GraphQL schema according to the
    [GraphQL specification](https://spec.graphql.org/).

**Explanatory Text**

Before composition, every individual source schema must be valid as per the
official GraphQL specification. Common reasons a schema may be considered
"invalid GraphQL" include:

- **Syntax Errors**: Missing braces, invalid tokens, or misplaced punctuation.
- **Unknown Types**: Referencing types that are not defined within the schema or
  imported from elsewhere.
- **Invalid Directive Usage**: Omitting required arguments to directives or
  using directives in disallowed locations.
- **Invalid Default Values**: Providing default values for arguments or fields
  that do not conform to the type (e.g., a default of `null` for a non-null
  field, an invalid enum value, etc.).
- **Conflicting Type Definitions**: Defining or overriding a built-in type or
  directive incorrectly.

When any of these validation checks fail for a particular source schema, that
schema does not meet the baseline requirements for composition, and the
composition process cannot proceed. An `INVALID_GRAPHQL` error is raised,
prompting the schema owner to correct the GraphQL violations before retrying
composition.

**Examples**

In the following counter-example, the schema is invalid because the type `User`
is referenced in the `Query` type but never defined:

```graphql counter-example
type Query {
  user: User
}

# The type "User" is never defined; this is invalid GraphQL.
```

In this counter-example, `"INVALID_VALUE"` is not a valid `Role`, causing
`INVALID_GRAPHQL`.

```graphql counter-example
enum Role {
  ADMIN
  USER
}

type Query {
  users(role: Role = "INVALID_VALUE"): [String]
}
```

The GraphQL spec requires all non-null directive arguments to be supplied. The
omission of the `fields` argument in the `@provides` directive triggers
`INVALID_GRAPHQL`.

```graphql counter-example
directive @provides(fields: String!) on FIELD_DEFINITION

type Product {
  price: Float @provides
  # "fields" argument is required, but not provided.
}
```

### Override Collision with Another Directive

**Error Code**  
`OVERRIDE_COLLISION_WITH_ANOTHER_DIRECTIVE`

**Severity**  
ERROR

**Formal Specification**

- Let {schemas} be the set of all source schemas to be composed.
- For each {schema} in {schemas}:
  - Let {types} be the set of all composite types in {schema}.
  - For each {type} in {types}:
    - Let {fields} be the set of fields on {type}.
    - For each {field} in {fields}:
      - If {field} is annotated with `@override`:
        - {field} must **not** be annotated with `@external`

**Explanatory Text**

The `@override` directive designates that ownership of a field is transferred
from one source schema to another in the resulting composite schema. When such a
transfer occurs, that field **cannot** also be annotated `@external`. A field
declared as `@external` is originally defined in a **different** source schema.
Overriding a field and simultaneously claiming it is external to the local
schema is contradictory.

In this case composition fails with an
`OVERRIDE_COLLISION_WITH_ANOTHER_DIRECTIVE` error.

**Examples**

In this scenario, `User.fullName` is defined in **Schema A** but overridden in
**Schema B**. Since `@override` is **not** combined with any of `@external` on
the same field, no collision occurs.

```graphql example
# Source Schema A
type User {
  id: ID!
  fullName: String
}

# Source Schema B
type User {
  id: ID!
  fullName: String @override(from: "SchemaA")
}
```

Here, `amount` is marked with both `@override` and `@external`. This violates
the rule because the field is simultaneously labeled as “override from another
schema” and “external” in the local schema, producing an
`OVERRIDE_COLLISION_WITH_ANOTHER_DIRECTIVE` error.

```graphql counter-example
# Source Schema A
type Payment {
  id: ID!
  amount: Int
}

# Source Schema B
type Payment {
  id: ID!
  amount: Int @override(from: "SchemaA") @external
}
```

### Override from Self Error

**Error Code**  
`OVERRIDE_FROM_SELF_ERROR`

**Severity**  
ERROR

**Formal Specification**

- Let {schemas} be the set of all source schemas to be composed.
- For each {schema} in {schemas}:
  - Let {types} be the set of all composite types in {schema}.
  - For each {type} in {types}:
    - Let {fields} be the set of fields on {type}.
    - For each {field} in {fields}:
      - If {field} is annotated with `@override`:
        - Let {from} be the value of the `from` argument of the `@override`
          directive on {field}.
        - {from} must **not** be the same as the name of {schema}:

**Explanatory Text**

When using `@override`, the `from` argument indicates the name of the source
schema that originally owns the field. Overriding from the **same** schema
creates a contradiction, as it implies both local and transferred ownership of
the field within one schema. If the `from` value matches the local schema name,
it triggers an `OVERRIDE_FROM_SELF_ERROR`.

**Examples**

In the following example, **Schema B** overrides the field `amount` from
**Schema A**. The two schema names are different, so no error is raised.

```graphql example
# Source Schema A
type Bill {
  id: ID!
  amount: Int
}

# Source Schema B
type Bill {
  id: ID!
  amount: Int @override(from: "SchemaA")
}
```

In the following counter-example, the local schema is also `"SchemaA"`, and the
`from` argument is `"SchemaA"`. Overriding a field from the same schema is not
allowed, causing an `OVERRIDE_FROM_SELF_ERROR`.

```graphql counter-example
# Source Schema A (named "SchemaA")
type Bill {
  id: ID!
  amount: Int @override(from: "SchemaA")
}
```

### Override on Interface

**Error Code**  
`OVERRIDE_ON_INTERFACE`

**Severity**  
ERROR

**Formal Specification**

- Let {schemas} be the set of all source schemas to be composed.
- For each {schema} in {schemas}:
  - Let {types} be the set of all interface types in {schema}.
  - For each {type} in {types}:
    - Let {fields} be the set of fields on {type}.
    - For each {field} in {fields}:
      - {field} must **not** be annotated with `@override`

**Explanatory Text**

The `@override` directive designates that ownership of a field is transferred
from one source schema to another. In the context of interface types, fields are
abstract—objects that implement the interface are responsible for providing the
actual fields. Consequently, it is invalid to attach `@override` directly to an
interface field. Doing so leads to an `OVERRIDE_ON_INTERFACE` error because
there is no concrete field implementation on the interface itself that can be
overridden.

**Examples**

In this valid example, `@override` is used on a field of an object type,
ensuring that the field definition is concrete and can be reassigned to another
schema.

Since `@override` is **not** used on any interface fields, no error is produced.

```graphql example
# Source Schema A
type Order {
  id: ID!
  amount: Int
}

# Source Schema B
type Order {
  id: ID!
  amount: Int @override(from: "SchemaA")
}
```

In the following counter-example, `Bill.amount` is declared on an **interface**
type and annotated with `@override`. This violates the rule because the
interface field itself is not eligible for ownership transfer. The composition
fails with an `OVERRIDE_ON_INTERFACE` error.

```graphql counter-example
# Source Schema A
interface Bill {
  id: ID!
  amount: Int @override(from: "SchemaB")
}
```

### Override Source Has Override

**Error Code**  
`OVERRIDE_SOURCE_HAS_OVERRIDE`

**Severity**  
ERROR

**Formal Specification**

- Let {schemas} be the set of all source schemas to be composed.
- Let {groupedTypes} be a map grouping all object types from {schemas} by their
  type name.
- For each {typeGroup} in {groupedTypes}:
  - Let {types} be the set of object types in {typeGroup}.
  - Let {groupedFields} be a map grouping every field across all {types} by
    their field name.
  - For each {fieldGroup} in {groupedFields}:
    - Let {fields} be the set of field definitions in {fieldGroup}.
    - Let {overrides} be the list of `@override` directives present among those
      {fields}.
    - If {overrides} has fewer than 2 elements:
      - Continue
    - Let {firstOverride} be the first directive in {overrides}.
    - Let {from} be the value of the `from` argument on {firstOverride}.
    - Let {sourceSchema} be the schema definining {firstOverride}.
    - Let {visited} be an empty set.
    - Add {sourceSchema} to {visited}.
    - While {from} is not null:
      - {from} must **not** be in {visited}.
      - Add {from} to {visited}.
      - Let {sourceField} be the field in {fields} that belongs to the schema
        named {from}.
      - If {sourceField} does not exist:
        - Break
      - If {sourceField} is **not** annotated with `@override`:
        - Break
      - Let {from} be the value of the `from` argument on that `@override`
        directive.
    - The size of {visited} must be equal to the size of {overrides}.

**Explanatory Text**

A field marked with `@override` signifies that its ownership is being taken over
by another schema. If multiple schemas try to override the same field, or if the
ownership chain loops back on itself, the composed schema has more than one
`@override` for a single field. This creates ambiguity about which schema
ultimately owns that field.

Hence, **only one** `@override` may ever apply to a particular field across all
source schemas. Attempting multiple overrides, or forming any cycle of overrides
for the same field, triggers the `OVERRIDE_SOURCE_HAS_OVERRIDE` error.

**Examples**

In this scenario, `Bill.amount` is originally owned by **Schema A** but is
overridden in **Schema B**. No other schema further attempts to override the
same field, so the composition is valid.

```graphql example
# Source Schema A
type Bill {
  id: ID!
  amount: Int
}

# Source Schema B
type Bill {
  id: ID!
  amount: Int @override(from: "SchemaA")
}
```

Here, **Schema A** overrides `Bill.amount` from **Schema B**, while **Schema B**
also overrides the same field from **Schema A**. This circular override makes it
impossible to discern a single “owner” of the field `Bill.amount`, raising an
`OVERRIDE_SOURCE_HAS_OVERRIDE` error.

```graphql counter-example
# Source Schema A (named "SchemaA")
type Bill {
  id: ID!
  amount: Int @override(from: "SchemaB")
}

# Source Schema B (named "SchemaB")
type Bill {
  id: ID!
  amount: Int @override(from: "SchemaA")
}
```

In this case, the same field `Bill.amount` is overridden successively by A, then
B, then C. Tracing these overrides forms a cycle (A → B → C → A). This again
produces an `OVERRIDE_SOURCE_HAS_OVERRIDE` error.

```graphql counter-example
# Source Schema A (named "A")
type Bill {
  id: ID!
  amount: Int @override(from: "B")
}

# Source Schema B (named "B")
type Bill {
  id: ID!
  amount: Int @override(from: "C")
}

# Source Schema C (named "C")
type Bill {
  id: ID!
  amount: Int @override(from: "A")
}
```

In the following counter-example, the field `Bill.amount` is overridden by
multiple schemas. The overrides do not form a cycle, hence there are multiple
overrides for the same field, triggering an `OVERRIDE_SOURCE_HAS_OVERRIDE`
error.

```graphql counter-example
# Source Schema A
type Bill {
  id: ID!
  amount: Int @override(from: "SchemaC")
}

# Source Schema B
type Bill {
  id: ID!
  amount: Int @override(from: "SchemaC")
}

# Source Schema C
type Bill {
  id: ID!
  amount: Int
}
```

### External Collision with Another Directive

**Error Code**  
`EXTERNAL_COLLISION_WITH_ANOTHER_DIRECTIVE`

**Severity**  
ERROR

**Formal Specification**

- Let {schemas} be the set of all source schemas to be composed.
- For each {schema} in {schemas}:
  - Let {types} be the set of all composite types in {schema}.
  - For each {type} in {types}:
    - Let {fields} be the set of fields on {type}.
    - For each {field} in {fields}:
      - If {field} is annotated with `@external`:
        - For each {argument} in {field}:
          - {argument} must **not** be annotated with `@require`
        - {field} must **not** be annotated with `@provides`

**Explanatory Text**

The `@external` directive indicates that a field is **defined** in a different
source schema, and the current schema merely references it. Therefore, a field
marked with `@external` must **not** simultaneously carry directives that assume
local ownership or resolution responsibility, such as:

- **`@provides`**: Declares that the field can supply additional nested fields
  from the local schema, which conflicts with the notion of an external field
  whose definition resides elsewhere.

- **`@require`**: Specifies dependencies on other fields to resolve this field.
  Since `@external` fields are not locally resolved, there is no need for
  `@require`.

- **`@override`**: Transfers ownership of the field’s definition from one schema
  to another, which is incompatible with an already-external field definition.
  Yet this behaviour is covered by the
  `OVERRIDE_COLLISION_WITH_ANOTHER_DIRECTIVE` rule.

Any combination of `@external` with either `@provides` or `@require` on the same
field results in inconsistent semantics. In such scenarios, an
`EXTERNAL_COLLISION_WITH_ANOTHER_DIRECTIVE` error is raised.

**Examples**

In this example, `method` is **only** annotated with `@external` in Schema B,
without any other directive. This usage is valid.

```graphql example
# Source Schema A
type Payment {
  id: ID!
  method: String
}

# Source Schema B
type Payment {
  id: ID!
  # This field is external, defined in Schema A.
  method: String @external
}
```

In this counter-example, `description` is annotated with `@external` and also
with `@provides`. Because `@external` and `@provides` cannot co-exist on the
same field, an `EXTERNAL_COLLISION_WITH_ANOTHER_DIRECTIVE` error is produced.

```graphql counter-example
# Source Schema A
type Invoice {
  id: ID!
  description: String
}

# Source Schema B
type Invoice {
  id: ID!
  description: String @external @provides(fields: "length")
}
```

The following example is invalid, since `title` is marked with both `@external`
and has an argument that is annotated with `@require`. This conflict leads to an
`EXTERNAL_COLLISION_WITH_ANOTHER_DIRECTIVE` error.

```graphql counter-example
# Source Schema A
type Book {
  id: ID!
  title: String
  subtitle: String
}

# Source Schema B
type Book {
  id: ID!
  title(subtitle: String @require(fields: "subtitle")) @external
}
```

### Key Invalid Fields Type

**Error Code**  
`KEY_INVALID_FIELDS_TYPE`

**Severity**  
ERROR

**Formal Specification**

- Let {schemas} be the set of all source schemas to be composed.
- For each {schema} in {schemas}:
  - Let {types} be the set of all composite types in {schema}.
  - For each {type} in {types}:
    - If {type} is annotated with `@key`:
      - Let {fieldsArg} be the value of the `fields` argument in the `@key`
        directive.
      - {fieldsArg} must be a string.

**Explanatory Text**

The `@key` directive designates the fields used to identify a particular object
uniquely. The `fields` argument accepts a **string** that represents a selection
set (for example, `"id"`, or `"id otherField"`). If the `fields` argument is
provided as any non-string type (e.g., `Boolean`, `Int`, `Array`), the schema
fails to compose correctly because it cannot parse a valid field selection.

**Examples**

In this example, the `@key` directive’s `fields` argument is the string
`"id uuid"`, identifying two fields that form the object key. This usage is
valid.

```graphql example
type User @key(fields: "id uuid") {
  id: ID!
  uuid: ID!
  name: String
}

type Query {
  users: [User]
}
```

Here, the `fields` argument is provided as a boolean (`true`) instead of a
string. This violates the directive requirement and triggers a
`KEY_INVALID_FIELDS_TYPE` error.

```graphql counter-example
type User @key(fields: true) {
  id: ID
}
```

### Provides Invalid Fields Type

**Error Code**  
`PROVIDES_INVALID_FIELDS_TYPE`

**Severity**  
ERROR

**Formal Specification**

- Let {schemas} be the set of all source schemas to be composed.
- For each {schema} in {schemas}:
  - Let {types} be the set of all composite types in {schema}.
  - For each {type} in {types}:
    - Let {fields} be the set of fields on {type}.
    - For each {field} in {fields}:
      - If {field} is annotated with `@provides`:
        - Let {fieldsArg} be the value of the `fields` argument on the
          `@provides` directive.
        - {fieldsArg} must be a string.

**Explanatory Text**

The `@provides` directive indicates that a field is **providing** one or more
additional fields on the returned (child) type. The `fields` argument accepts a
**string** representing a GraphQL selection set (for example, `"title author"`).
If the `fields` argument is given as a non-string type (e.g., `Boolean`, `Int`,
`Array`), the schema fails to compose because it cannot interpret a valid
selection set.

**Examples**

In this valid example, the `@provides` directive on `details` uses the string
`"features specifications"` to specify that both fields are provided in the
child type `ProductDetails`.

```graphql example
type Product {
  id: ID!
  details: ProductDetails @provides(fields: "features specifications")
}

type ProductDetails {
  features: [String]
  specifications: String
}

type Query {
  products: [Product]
}
```

Here, the `@provides` directive includes a numeric value (`123`) instead of a
string in its `fields` argument. This invalid usage raises a
`PROVIDES_INVALID_FIELDS_TYPE` error.

```graphql counter-example
type Product {
  id: ID!
  details: ProductDetails @provides(fields: 123)
}

type ProductDetails {
  features: [String]
  specifications: String
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
