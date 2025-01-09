# Schema Composition

The schema composition describes the process of merging multiple source schemas
into a single GraphQL schema, known as the _composite execution schema_, which
is a valid GraphQL schema annotated with execution directives. This composite
execution schema is the output of the schema composition process. The schema
composition process is divided into three main steps: **Validate Source
Schemas**, **Merge Source Schemas**, and **Validate Satisfiability**, which are
run in sequence to produce the composite execution schema.

Although this chapter describes schema composition as a sequence of phases, an
implementation is not required to implement these steps exactly as presented.
Implementations may interleave or reorder the specified checks, or introduce
additional processing stages, provided that the final composed schema complies
with the requirements set forth in this specification. The composition rules and
resulting schema must remain consistent, but the specific structure or timing of
each validation step is left to the implementer.

## Validate Source Schemas

In this phase, each source schema is validated in isolation to ensure that it
satisfies the GraphQL specification and composition requirements. No
cross-schema references are considered here. Each source schema must have valid
syntax, well-formed type definitions, and correct directive usage. If any source
schema fails these checks, composition does not proceed.

## Merge Source Schemas

Once all source schemas have passed individual validation, they are merged into
a single composite schema. This merging process is subdivided into three stages:
pre-merge validation, merge, and post-merge validation.

### Pre Merge Validation

Prior to merging the schemas, additional validations are performed that require
visibility into all source schemas but treat them as separate entities. This
step detects conflicts such as incompatible fields or default argument values
that would render the merged schema unusable. Detecting such conflicts early
prevents errors that would otherwise be discovered during the merge process.

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

#### Root Mutation Used

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

#### Root Query Used

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

#### Root Subscription Used

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

#### Key Fields Select Invalid Type

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

#### Key Directive in Fields Argument

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

#### Key Fields Has Arguments

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

#### Key Invalid Syntax

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
argument string is syntactically incorrect-missing closing braces, containing
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

#### Key Invalid Fields

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
references within that selection set must also refer to **actual** fields on the
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

#### Provides Directive in Fields Argument

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

#### Provides Fields Has Arguments

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

#### Provides Fields Missing External

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
additional fields belonging to the return type in this execution-specific path.
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

#### Query Root Type Inaccessible

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
violated. `QUERY_ROOT_TYPE_INACCESSIBLE` is raised because a schema's root query
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

#### Require Directive in Fields Argument

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
argument depends on in order to resolve the annotated field. When using
`@require(fields: "…")`, the `fields` argument must be a valid selection set
string **without** any additional directive applications. Applying a directive
(e.g., `@lowercase`) inside this selection set is not supported and triggers the
`REQUIRE_DIRECTIVE_IN_FIELDS_ARG` error.

**Examples**

In this valid usage, the `@require` directive's `fields` argument references
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

#### Require Invalid Fields Type

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

In the following example, the `@require` directive's `fields` argument is a
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

#### Require Invalid Syntax

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

The `@require` directive's `fields` argument must be syntactically valid
GraphQL. If the selection map string is malformed (e.g., missing closing braces,
unbalanced quotes, invalid tokens), then the schema cannot be composed
correctly. In such cases, the error `REQUIRE_INVALID_SYNTAX` is raised.

**Examples**

In the following example, the `@require` directive's `fields` argument is a
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

In the following counter-example, the `@require` directive's `fields` argument
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

#### Type Definition Invalid

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
purposes and must adhere to the specification's definitions. For example,
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

#### Type Kind Mismatch

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

#### Provides Invalid Syntax

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

The `@provides` directive's `fields` argument must be a syntactically valid
selection set string, as if you were selecting fields in a GraphQL query. If the
selection set is malformed (e.g., missing braces, unbalanced quotes, or invalid
tokens), the schema composition fails with a `PROVIDES_INVALID_SYNTAX` error.

**Examples**

Here, the `@provides` directive's `fields` argument is a valid selection set:

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

#### Invalid GraphQL

**Error Code**

`INVALID_GRAPHQL`

**Severity**

ERROR

**Formal Specification**

- Let {schemas} be the set of all source schemas to be composed.
- For Each {schema} in {schemas}
  - {schema} must be a syntactically valid
  - {schema} must be a semantically valid GraphQL schema according to the
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

#### Override Collision with Another Directive

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

#### Override from Self

**Error Code**

`OVERRIDE_FROM_SELF`

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
it triggers an `OVERRIDE_FROM_SELF` error.

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
allowed, causing an `OVERRIDE_FROM_SELF` error.

```graphql counter-example
# Source Schema A (named "SchemaA")
type Bill {
  id: ID!
  amount: Int @override(from: "SchemaA")
}
```

#### Override on Interface

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

#### Override Source Has Override

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
    - Let {sourceSchema} be the schema defining {firstOverride}.
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

#### External Collision with Another Directive

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

- **`@override`**: Transfers ownership of the field's definition from one schema
  to another, which is incompatible with an already-external field definition.
  Yet this is covered by the `OVERRIDE_COLLISION_WITH_ANOTHER_DIRECTIVE` rule.

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

#### Key Invalid Fields Type

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

In this example, the `@key` directive's `fields` argument is the string
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

#### Provides Invalid Fields Type

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

#### Provides on Non-Composite Field

**Error Code**

`PROVIDES_ON_NON_COMPOSITE_FIELD`

**Severity**

ERROR

**Formal Specification**

- Let {schemas} be the set of all source schemas to be composed.
- For each {schema} in {schemas}:
  - Let {types} be the set of all object and interface types in {schema}.
  - For each {type} in {types}:
    - Let {fields} be the set of fields on {type}.
    - For each {field} in {fields}:
      - If {field} is annotated with `@provides`:
        - Let {fieldType} be the base return type of {field} (i.e., unwrapped of
          any `[ ]` or `!`).
        - {fieldType} must be a interface or object type.

**Explanatory Text**

The `@provides` directive allows a field to “provide” additional nested fields
on the composite type it returns. If a field's base type is not an object or
interface type (e.g., `String`, `Int`, `Boolean`, `Enum`, `Union`, or an `Input`
type), it cannot hold nested fields for `@provides` to select. Consequently,
attaching `@provides` to such a field is invalid and raises a
`PROVIDES_ON_NON_OBJECT_FIELD` error.

**Examples**

Here, `profile` has an **object** base type `Profile`. The `@provides` directive
can validly specify sub-fields like `settings { theme }`.

```graphql example
type Profile {
  email: String
  settings: Settings
}

type Settings {
  notificationsEnabled: Boolean
  theme: String
}

type User {
  id: ID!
  profile: Profile @provides(fields: "settings { theme }")
}
```

In this counter-example, `email` has a scalar base type (`String`). Because
scalars do not expose sub-fields, attaching `@provides` to `email` triggers a
`PROVIDES_ON_NON_OBJECT_FIELD` error.

```graphql counter-example
type User {
  id: ID!
  email: String @provides(fields: "length")
}
```

#### External on Interface

**Error Code**

`EXTERNAL_ON_INTERFACE`

**Severity**

ERROR

**Formal Specification**

- Let {schemas} be the set of all source schemas to be composed.
- For each {schema} in {schemas}:
  - Let {types} be the set of all composite types in {schema}.
  - For each {type} in {types}:
    - If {type} is an interface type:
      - Let {fields} be the set of fields on {type}.
      - For each {field} in {fields}:
        - {field} must **not** be annotated with `@external`

**Explanatory Text**

The `@external` directive indicates that a field is **defined** and **resolved**
elsewhere, not in the current schema. In the case of an **interface** type,
fields are **abstract** - they do not have direct resolutions at the interface
level. Instead, each implementing object type provides the concrete field
implementations. Marking an **interface** field with `@external` is therefore
nonsensical, as there is no actual field resolution in the interface itself to
“borrow” from another schema. Such usage raises an `EXTERNAL_ON_INTERFACE`
error.

**Examples**

Here, the interface `Node` merely describes the field `id`. Object types `User`
and `Product` implement and resolve `id`. No `@external` usage occurs on the
interface itself, so no error is triggered.

```graphql example
interface Node {
  id: ID!
}

type User implements Node {
  id: ID!
  name: String
}

type Product implements Node {
  id: ID!
  price: Int
}
```

Since `id` is declared on an **interface** and marked with `@external`, the
composition fails with `EXTERNAL_ON_INTERFACE`. An interface does not own the
concrete field resolution, so it is invalid to mark any of its fields as
external.

```graphql counter-example
interface Node {
  id: ID! @external
}
```

#### `@lookup` Should Have Nullable Return Type

**Error Code**

`LOOKUP_SHOULD_HAVE_NULLABLE_RETURN_TYPE`

**Severity**

WARNING

**Formal Specification**

- Let {fields} be the set of all field definitions annotated with `@lookup` in
  the schema.
- For each {field} in {fields}:
  - Let {type} be the return type of {field}.
  - {type} must be a nullable type.

**Explanatory Text**

Fields annotated with the `@lookup` directive are intended to retrieve a single
entity based on provided arguments. To properly handle cases where the requested
entity does not exist, such fields should have a nullable return type. This
allows the field to return `null` when an entity matching the provided criteria
is not found, following the standard GraphQL practices for representing missing
data.

In a distributed system, it is likely that some entities will not be found on
other schemas, even when those schemas contribute fields to the type. Ensuring
that `@lookup` fields have nullable return types also avoids GraphQL errors on
schemas and prevents result erasure through non-null propagation. By allowing
null to be returned when an entity is not found, the system can gracefully
handle missing data without causing exceptions or unexpected behavior.

Ensuring that `@lookup` fields have nullable return types allows gateways to
distinguish between cases where an entity is not found (receiving null) and
other error conditions that may have to be propagated to the client.

For example, the following usage is recommended:

```graphql example
extend type Query {
  userById(id: ID!): User @lookup
}

type User {
  id: ID!
  name: String
}
```

In this example, `userById` returns a nullable `User` type, aligning with the
recommendation.

**Examples**

This counter-example demonstrates an invalid usage:

```graphql counter-example
extend type Query {
  userById(id: ID!): User! @lookup
}

type User {
  id: ID!
  name: String
}
```

Here, `userById` returns a non-nullable `User!`, which does not align with the
recommendation that a `@lookup` field should have a nullable return type.

#### `@lookup` must not return a list

**Error Code**

`LOOKUP_MUST_NOT_RETURN_LIST`

**Severity** ERROR

**Formal Specification**

- Let {fields} be the set of all field definitions annotated with `@lookup` in
  the schema.
- For each {field} in {fields}:
  - Let {type} be the return type of {field}.
  - {IsListType(type)} must be false.

IsListType(type):

- If {type} is a Non-Null type:
  - Let {innerType} be the inner type of {type}.
  - Return {IsListType(innerType)}.
- Else if {type} is a List type:
  - Return true.
- Else:
  - Return false.

**Explanatory Text**

Fields annotated with the `@lookup` directive are intended to retrieve a single
entity based on provided arguments. To avoid ambiguity in entity resolution,
such fields must return a single object and not a list. This validation rule
enforces that any field annotated with `@lookup` must have a return type that is
**NOT** a list.

**Examples**

For example, the following usage is valid:

```graphql example
extend type Query {
  userById(id: ID!): User @lookup
}

type User {
  id: ID!
  name: String
}
```

In this example, `userById` returns a `User` object, satisfying the requirement.

This counter-example demonstrates an invalid usage:

```graphql counter-example
extend type Query {
  usersByIds(ids: [ID!]!): [User!] @lookup
}

type User {
  id: ID!
  name: String
}
```

Here, `usersByIds` returns a list of `User` objects, which violates the
requirement that a `@lookup` field must return a single object.

#### Input Field Default Mismatch

**Error Code**

`INPUT_FIELD_DEFAULT_MISMATCH`

**Formal Specification**

- Let {inputFieldsByName} be a map where the key is the name of an input field
  and the value is a list of input fields from different source schemas from the
  same type with the same name.
- For each {inputFields} in {inputFieldsByName}:
  - Let {defaultValues} be a set containing the default values of each input
    field in {inputFields}.
  - If the size of {defaultValues} is greater than 1:
    - {InputFieldsHaveConsistentDefaults(inputFields)} must be false.

InputFieldsHaveConsistentDefaults(inputFields):

- Given each pair of input fields {inputFieldA} and {inputFieldB} in
  {inputFields}:
  - If the default value of {inputFieldA} is not equal to the default value of
    {inputFieldB}:
    - return false
- return true

**Explanatory Text**

Input fields in different source schemas that have the same name are required to
have consistent default values. This ensures that there is no ambiguity or
inconsistency when merging input fields from different source schemas.

A mismatch in default values for input fields with the same name across
different source schemas will result in a schema composition error.

**Examples**

In the the following example both source schemas have an input field `field1`
with the same default value. This is valid:

```graphql example
# Schema A

input BookFilter {
  genre: Genre = FANTASY
}

enum Genre {
  FANTASY
  SCIENCE_FICTION
}

# Schema B
input BookFilter {
  genre: Genre = FANTASY
}

enum Genre {
  FANTASY
  SCIENCE_FICTION
}
```

In the following example both source schemas define an input field
`minPageCount` with different default values. This is invalid:

```graphql counter-example
# Schema A

input BookFilter {
  minPageCount: Int = 10
}

# Schema B

input BookFilter {
  minPageCount: Int = 20
}
```

#### Input Field Types mergeable

**Error Code**

`INPUT_FIELD_TYPES_NOT_MERGEABLE`

**Formal Specification**

- Let {fieldsByName} be a map of field lists where the key is the name of a
  field and the value is a list of fields from mergeable input types from
  different source schemas with the same name.
- For each {fields} in {fieldsByName}:
  - {InputFieldsAreMergeable(fields)} must be true.

InputFieldsAreMergeable(fields):

- Given each pair of members {fieldA} and {fieldB} in {fields}:
  - Let {typeA} be the type of {fieldA}.
  - Let {typeB} be the type of {fieldB}.
  - {SameTypeShape(typeA, typeB)} must be true.

**Explanatory Text**

The input fields of input objects with the same name must be mergeable. This
rule ensures that input objects with the same name in different source schemas
have fields that can be merged consistently without conflicts.

Input fields are considered mergeable when they share the same name and have
compatible types. The compatibility of types is determined by their structure
(e.g., lists), excluding nullability. Mergeable input fields with different
nullability are considered mergeable, and the resulting merged field will be the
most permissive of the two.

In this example, the field `name` in `AuthorInput` has compatible types across
source schemas, making them mergeable:

```graphql example
input AuthorInput {
  name: String!
}

input AuthorInput {
  name: String
}
```

The following example shows that fields are mergeable if they have different
nullability but the named type is the same and the list structure is the same.

```graphql example
input AuthorInput {
  tags: [String!]
}

input AuthorInput {
  tags: [String]!
}

input AuthorInput {
  tags: [String]
}
```

In this example, the field `birthdate` on `AuthorInput` is not mergeable as the
field has different named types (`String` and `DateTime`) across source schemas:

```graphql counter-example
input AuthorInput {
  birthdate: String!
}

input AuthorInput {
  birthdate: DateTime!
}
```

#### Enum Values Mismatch

**Error Code**

`ENUM_VALUES_MISMATCH`

**Formal Specification**

- Let {enumNames} be the set of all enum type names across all source schemas.
- For each {enumName} in {enumNames}:
  - Let {enums} be the list of all enum types from different source schemas with
    the name {enumName}.
  - {EnumsAreMergeable(enums)} must be true.

EnumsAreMergeable(enums):

- If {enums} has fewer than 2 elements:
  - Return true.
- Let {inaccessibleValues} be the set of values that are declared as
  `@inaccessible` in {enums}.
- Let {requiredValues} be the set of values in {enums} that are not in
  {inaccessibleValues}.
- For each {enum} in {enums}
  - Let {enumValues} be the set of all values of {enum} that are not in
    {inaccessibleValues}.
  - {requiredValues} must be equal to {enumValues}

**Explanatory Text**

This rule ensures that enum types with the same name across different source
schemas in a composite schema have identical sets of values. Enums must be
consistent across source schemas to avoid conflicts and ambiguities in the
composite schema.

When an enum is defined with differing values, it can lead to confusion and
errors in query execution. For instance, a value valid in one schema might be
passed to another where it's unrecognized, leading to unexpected behavior or
failures. This rule prevents such inconsistencies by enforcing that all
instances of the same named enum across schemas have an exact match in their
values.

In this example, both source schemas define `Genre` with the same value
`FANTASY`, satisfying the rule:

```graphql example
enum Genre {
  FANTASY
}

enum Genre {
  FANTASY
}
```

Here, the two definitions of `Genre` have different values (`FANTASY` and
`SCIENCE_FICTION`), violating the rule:

```graphql counter-example
enum Genre {
  FANTASY
}

enum Genre {
  SCIENCE_FICTION
}
```

Here, the two definitions of `Genre` have shared values and additional values
declared as `@inaccessible`, satisfying the rule:

```graphql example
enum Genre {
  FANTASY
  SCIENCE_FICTION @inaccessible
}

enum Genre {
  FANTASY
}
```

#### Input With Missing Required Fields

**Error Code:**

`INPUT_WITH_MISSING_REQUIRED_FIELDS`

**Severity:**

ERROR

**Formal Specification:**

- Let {typeNames} be the set of all input object types names from all source
  schemas that are not declared as `@inaccessible`.
- For each {typeName} in {typeNames}:
  - Let {types} be the list of all input object types from different source
    schemas with the name {typeName}.
  - {AreTypesConsistent(types)} must be true.

AreTypesConsistent(inputs):

- Let {requiredFields} be the intersection of all field names across all input
  objects in {inputs} that are not marked as `@inaccessible` in any schema and
  have a non-nullable type in at least one schema.
- For each {input} in {inputs}:
  - For each {requiredField} in {requiredFields}:
    - If {requiredField} is not in {input}:
      - Return false

**Explanatory Text:**

Input types are merged by intersection, meaning that the merged input type will
have all fields that are present in all input types with the same name. This
rule ensures that input object types with the same name across different schemas
share a consistent set of required fields.

**Examples**

If all schemas define `BookFilter` with the required field `title`, the rule is
satisfied:

```graphql
# Schema A
input BookFilter {
  title: String!
  author: String
}

# Schema B
input BookFilter {
  title: String!
  yearPublished: Int
}
```

If `title` is required in one source schema but missing in another, this
violates the rule:

```graphql
# Schema A
input BookFilter {
  title: String!
  author: String
}

# Schema B
input BookFilter {
  author: String
  yearPublished: Int
}
```

In this invalid case, `title` is mandatory in Schema A but not defined in Schema
B, causing inconsistency in required fields across schemas.

#### Field Argument Types Mergeable

**Error Code**

`FIELD_ARGUMENT_TYPES_NOT_MERGEABLE`

**Severity**

ERROR

**Formal Specification**

- Let {typeNames} be the set of all output type names from all source schemas.
- For each {typeName} in {typeNames}
  - Let {types} be the set of all types with the {typeName} from all source
    schemas.
  - Let {fieldNames} be the set of all field names from all {types}.
  - For each {fieldName} in {fieldNames}
    - Let {fields} be the set of all fields with the {fieldName} from all
      {types}.
    - For each {field} in {fields}
      - Let {argumentNames} be the set of all argument names from all {fields}.
      - For each {argumentName} in {argumentNames}
        - Let {arguments} be the set of all arguments with the {argumentName}
          from all {fields}.
        - For each pair of {argumentA} and {argumentB} in {arguments}
          - {ArgumentsAreMergeable(argumentA, argumentB)} must be true.

ArgumentsAreMergeable(argumentA, argumentB):

- Let {typeA} be the type of {argumentA}
- Let {typeB} be the type of {argumentB}
- {InputTypesAreMergeable(typeA, typeB)} must be true.

**Explanatory Text**

When multiple schemas define the same field name on the same output type (e.g.,
`User.field`), these fields can be merged if their arguments are compatible.
Compatibility extends not only to the output field types themselves, but to each
argument's input type as well. The schemas must agree on each argument's name
and have compatible types, so that the composed schema can unify the definitions
into a single consistent field specification.

_Nullability_

Different nullability requirements on arguments are still considered mergeable.
For example, if one schema accepts `String!` and the other accepts `String`,
these schemas can merge; the resulting argument type typically adopts the least
restrictive (nullable) version.

_Lists_ Lists of different nullability (e.g., `[String!]` vs. `[String]!` vs.
`[String]`) remain mergeable as long as they otherwise refer to the same inner
type. Essentially, the same principle of “least restrictive” nullability merges
them successfully.

_Incompatible Types_

If argument types differ on the named type itself - for example, one uses
`String` while the other uses `DateTime` - this causes a
`FIELD_ARGUMENT_TYPES_NOT_MERGEABLE` error. Similarly, if one schema has
`[String]` but another has `[DateTime]`, they are incompatible.

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

During this stage, all definitions from each source schema are combined into a
single schema. This section defines the rules for merging schema definitions.
The goal is to create a composite schema that includes all type system members
from each source schema that are publicly accessible.

MergeSchemas(schemas):

- Let {mergedSchema} be an empty schema.
- Let {memberNames} be the set of all object, interface, union, enum and input
  type names in {schemas}.
- For each {memberName} in {memberNames}:
  - Let {types} be the set of all types named {memberName} across all source
    schemas.
  - Let {mergedType} be the result of {MergeTypes(types)}.
  - If {mergedType} is not {null}:
    - Add {mergedType} to {mergedSchema}.
- Return {mergedSchema}.

MergeTypes(types):

- Let {firstType} be the first type in {types}.
- Let {kind} be the kind of {firstType}.
- Assert: All types in {types} have the same kind.
- If {kind} is `SCALAR`:
  - Return the result of {MergeScalarTypes(types)}.
- If {kind} is `INTERFACE`:
  - Return the result of {MergeInterfaceTypes(types)}.
- If {kind} is `ENUM`:
  - Return the result of {MergeEnumTypes(types)}.
- If {kind} is `UNION`:
  - Return the result of {MergeUnionTypes(types)}.
- If {kind} is `INPUT_OBJECT`:
  - Return the result of {MergeInputTypes(types)}.
- If {kind} is `OBJECT`:
  - Return the result of {MergeObjectTypes(types)}.

#### Merge Scalar Types

**Formal Specification**

MergeScalarTypes(scalars):

- If any {scalar} in {scalars} is marked with `@inaccessible`
  - Return {null}
- Let {firstScalar} be the first scalar in {scalars}.
- Let {description} be the description of {firstScalar}.
- For each {scalar} in {scalars}:
  - If {description} is {null}:
    - Set {description} to the description of {scalar}.
- Return a new scalar type with the name of {firstScalar} and description of
  {description}.

**Explanatory Text**

{MergeScalarTypes(scalars)} merges multiple scalar definitions that share the
same name into a single scalar type. It filters out scalars marked with
`@inaccessible` and unifies descriptions so that the final type retains the
first available non-`null` description.

_Inaccessible Scalars_

If any scalar is labeled with `@inaccessible`, the merge immediately returns
`null`. A scalar that cannot be exposed to consumers renders the entire type
unusable.

_Combining Descriptions_

The final description is determined by the first non-`null` description found in
the list of scalars. If no descriptions are found, the final description is
`null`.

**Examples**

Here, two `Date` scalar types from different schemas are merged into a single
composed `Date` scalar type.

```graphql example
# Schema A

scalar Date

# Schema B

"A scalar representing a calendar date."
scalar Date

# Composed Result

"A scalar representing a calendar date."
scalar Date
```

#### Merge Interface Types

**Formal Specification**

MergeInterfaceTypes(types):

- If any {type} in {types} is marked with `@inaccessible`
  - Return {null}
- Let {firstType} be the first type in {types}.
- Let {typeName} be the name of {firstType}.
- Let {description} be the description of {firstType}.
- Let {fields} be an empty set.
- For each {type} in {types}:
  - If {description} is {null}:
    - Set {description} to the description of {type}.
- Let {fieldNames} be the set of all field names in {types}.
- For each {fieldName} in {fieldNames}:
  - Let {field} be the set of fields with the name {fieldName} in {types}.
  - Let {mergedField} be the result of {MergeFieldDefinitions(fields)}.
  - If {mergedField} is not {null}:
    - Add {mergedField} to {fields}.
- Return a new interface type with the name of {typeName}, description of
  {description}, and fields of {fields}.

**Explanatory Text**

{MergeInterfaceTypes(types)} unifies multiple interface definitions (all sharing
the _same name_) into a single composed interface type. If any one of these
interfaces is marked `@inaccessible`, the merge immediately returns `null`,
preventing inclusion of that interface in the final schema.

_Inaccessible Interfaces_

A type marked `@inaccessible` disqualifies the entire merge, ensuring no
references to inaccessible types appear in the final schema.

_Combining Descriptions_

Among the valid interfaces, the description is taken from the first non-`null`
description encountered. If all interfaces lack a description, the resulting
interface has none.

_Merging Fields_

Each interface contributes its fields. Those fields that share the same name
across multiple interfaces are reconciled via {MergeFieldDefinitions(fields)}.
This ensures any differences in type, nullability, or other constraints are
resolved before appearing in the final interface.

By applying these steps, {MergeInterfaceTypes(types)} produces a coherent
interface type definition that reflects the fields from all compatible sources
while adhering to accessibility constraints.

**Examples**

Here, two `Product` interface types from different schemas are merged into a
single composed `Product` interface type.

```graphql example
# Schema A

interface Product {
  id: ID!
  name: String
}

# Schema B

interface Product {
  id: ID!
  createdAt: String
}

# Composed Result

interface Entity {
  id: ID!
  name: String
  createdAt: String
}
```

In this example, the `Product` interface type from two schemas is merged. The
`id` field is shared across both schemas, while `name` and `createdAt` fields
are contributed by the individual source schemas. The resulting composed type
includes all fields.

The following example shows how the description is retained when merging
interface types:

```graphql example
# Schema A

"""
First description
"""
interface Product {
  id: ID!
}

# Schema B

"""
Second description
"""
interface Product {
  id: ID!
}

# Composed Result

"""
First description
"""
interface Product {
  id: ID!
}
```

#### Merge Enum Types

**Formal Specification**

MergeEnumTypes(enums):

- If any {enum} in {enums} is marked with `@inaccessible`
  - Return {null}
- Let {firstEnum} be the first enum in {enums}.
- If {enums} contains only one enum
  - Return a new enum type with the name of {firstEnum}, description of
    {firstEnum}, and enum values of {firstEnum} excluding any marked with
    `@inaccessible`.
- Let {typeName} be the name of {firstEnum}.
- Let {description} be the description of {firstEnum}.
- Let {enumValues} be the set of all enum values in {enums}.
- For each {enum} in {enums}:
  - If {description} is {null}:
    - Set {description} to the description of {enum}.
  - For each {enumValue} in the enum values of {enum}:
    - If {enumValue} is marked with `@inaccessible`
      - Remove {enumValue} from {enumValues}.
- Return a new enum type with the name of {typeName}, description of
  {description}, and enum values of {enumValues}.

**Explanatory Text**

{MergeEnumTypes(enums)} consolidates multiple enum definitions (all sharing the
_same name_) into one final enum type, while filtering out any parts marked with
`@inaccessible`. If an entire enum is inaccessible, the merge returns `null`.

_Inaccessible Enums_

If any enum in the input set is marked `@inaccessible`, the entire merge
operation is invalid. The algorithm immediately returns `null`, since that type
cannot appear in the composed schema.

_Single vs. Multiple Enum Definitions_

When only one enum definition is present (after removing any inaccessible ones),
it is used as is, except that any values marked with `@inaccessible` are
excluded.

However, if an enum appears in multiple schemas, the enums must match exactly in
their values and structure unless some values are excluded using the
`@inaccessible` directive. This behavior is enforced by prior validation but is
important to note as it determines how mismatched enums are handled.

_Combining Descriptions_

The first non-`null` description encountered among the enums is used for the
final definition. If no definitions supply a description, the merged enum will
have none.

**Examples**

Here, two `Status` enums from different schemas are merged into a single
composed `Status` enum. The enums are identical, so the composed enum exactly
matches the source enums.

```graphql example
# Schema A

enum Status {
  ACTIVE
  INACTIVE
}

# Schema B

enum Status {
  ACTIVE
  INACTIVE
}

# Composed Result

enum Status {
  ACTIVE
  INACTIVE
}
```

If the enums differ in their values, the source schemas **must** define their
unique values as `@inaccessible` to exclude them from the composed enum.

```graphql example
# Schema A

enum Status {
  ACTIVE @inaccessible
  INACTIVE
}

# Schema B

enum Status {
  PENDING @inaccessible
  INACTIVE
}

# Composed Result

enum Status {
  INACTIVE
}
```

#### Merge Union Types

**Formal Specification**

MergeUnionTypes(unions):

- If any {union} in {unions} is marked with `@inaccessible`
  - Return {null}
- Let {firstUnion} be the first union in {unions}.
- Let {name} be the name of {firstUnion}.
- Let {description} be the description of {firstUnion}.
- Let {possibleTypes} be an empty set.
- For each {union} in {unions}:
  - If {description} is {null}:
    - Set {description} to the description of {union}.
  - For each {possibleType} in the possible types of {union}:
    - If {possibleType} is not marked with `@inaccessible` or `@internal`:
      - Add {possibleType} to {possibleTypes}.
- If {possibleTypes} is empty:
  - Return {null}
- Return a new union with the name of {name}, description of {description}, and
  possible types of {possibleTypes}.

**Explanatory Text**

{MergeUnionTypes(unions)} aggregates multiple union type definitions that share
the _same name_ into one unified union type. This process skips any union marked
with `@inaccessible` and excludes possible types marked with `@inaccessible` or
`@internal`.

_Inaccessible Unions_

If any union in the input list is marked `@inaccessible`, the merged result must
be `null` and cannot appear in the final schema.

_Combining Descriptions_

The first non-empty description that is found is used as the description for the
merged union. If no descriptions are found, the merged union will have no
description.

_Combining Possible Types_

Each union's possible types are considered in turn. Only those that are _not_
marked `@internal` or `@inaccessible` are included in the final composed union.
This preserves the valid types from all sources while systematically filtering
out anything inaccessible or intended for internal use only.

In case there are no possible types left after filtering, the merged union is
considered `@inaccessible` and cannot appear in the final schema.

**Examples**

Here, two `SearchResult` union types from different schemas are merged into a
single composed `SearchResult` type.

```graphql example
# Schema A

union SearchResult = Product | Order

# Schema B

union SearchResult = User | Order

# Composed Result

union SearchResult = Product | Order | User
```

In this example, the `SearchResult` union type from two schemas is merged. The
`Order` type is shared across both schemas, while `Product` and `User` types are
contributed by the individual source schemas. The resulting composed type
includes all valid possible types.

Another example shows how `@inaccessible` on a possible affects the merge:

```graphql example
# Schema A

union SearchResult = Product | Order

type Product @inaccessible {
  id: ID!
}

# Schema B

union SearchResult = User | Order

# Composed Result

union SearchResult = Order | User
```

In this case, the `Product` type is marked with `@inaccessible` in the first
schema. As a result, the `Product` type is excluded from the composed
`SearchResult`

#### Merge Input Types

**Formal Specification**

MergeInputTypes(types):

- If any {type} in {types} is marked with `@inaccessible`
  - Return {null}
- Let {firstType} be the first type in {types}.
- Let {typeName} be the name of {firstType}.
- Let {description} be the description of {firstType}.
- Let {fields} be an empty set.
- For each {type} in {types}:
  - If {description} is {null}:
    - Set {description} to the description of {type}.
- Let {fieldNames} be the set of all field names in {types}.
- For each {fieldName} in {fieldNames}:
  - Let {field} be the set of fields with the name {fieldName} in {types}.
  - Let {mergedField} be the result of {MergeInputField(fields)}.
  - If {mergedField} is not {null}:
    - Add {mergedField} to {fields}.

**Explanatory Text**

The {MergeInputTypes(types)} algorithm produces a single input type definition
by unifying multiple input types that share the _same name_. Each of these input
types may come from different sources, yet must align into one coherent
definition. Any type marked `@inaccessible` disqualifies the entire merge result
from inclusion in the composed schema.

_Inaccessible Types_

If an input type is annotated with `@inaccessible`, the algorithm immediately
returns `null`. Including an inaccessible type would mean exposing a field
that's not allowed in the composed schema.

_Combining Descriptions_

The first non-`null` description encountered is used for the final input type.
If no such description exists among the source types, the resulting input type
definition has no description.

_Merging Fields_

After filtering out inaccessible types, the algorithm merges each input field
name found across the remaining types. For each field, {MergeInputField(fields)}
is called to reconcile differences in type, nullability, default values, etc..
If a merged field ends up being `null` - for instance, because one of its
underlying definitions was inaccessible - that field is not included in the
final definition. The end result is a single input type that correctly unifies
every compatible field from the various sources.

**Examples**

Here, two `OrderInput` input types from different schemas are merged into a
single composed `OrderInput` type.

```graphql example
# Schema A

input OrderInput {
  id: ID!
  description: String
}

# Schema B

input OrderInput {
  id: ID!
  total: Float
}

# Composed Result

input OrderInput {
  id: ID!
  description: String
  total: Float
}
```

In this example, the `OrderInput` type from two schemas is merged. The `id`
field is shared across both schemas, while `description` and `total` fields are
contributed by the individual source schemas. The resulting composed type
includes all fields.

Another example demonstrates preserving descriptions during merging:

```graphql example
# Schema A

"""
First Description
"""
input OrderInput {
  id: ID!
}

# Schema B

"""
Second Description
"""
input OrderInput {
  id: ID!
}

# Composed Result

"""
First Description
"""
input OrderInput {
  id: ID!
}
```

In this case, the description from the first schema is retained, while the
fields are merged from both schemas to create the final `OrderInput` type.

#### Merge Object Types

**Formal Specification**

MergeObjectTypes(types):

- If any {type} in {types} is marked with `@inaccessible`
  - Return {null}
- Remove all types marked with `@internal` from {types}.
- Let {firstType} be the first type in {types}.
- Let {typeName} be the name of {firstType}.
- Let {description} be the description of {firstType}.
- Let {fields} be an empty set.
- For each {type} in {types}:
  - If {description} is {null}:
    - Set {description} to the description of {type}.
- Let {fieldNames} be the set of all field names in {types}.
- For each {fieldName} in {fieldNames}:
  - Let {field} be the set of fields with the name {fieldName} in {types}.
  - Let {mergedField} be the result of {MergeOutputField(fields)}.
  - If {mergedField} is not {null}:
    - Add {mergedField} to {fields}.
- Return a new object type with the name of {typeName}, description of
  {description}, fields of {fields}.

**Explanatory Text**

The {MergeObjectTypes(types)} algorithm combines multiple object type
definitions (all sharing the _same name_) into a single composed type. It
processes each candidate type, discarding any that are inaccessible or internal,
and then unifies their descriptions and fields.

_Inaccessible Types_

If an object type is marked with `@inaccessible`, the entire merged result must
be `null`; we cannot include that type in the composed schema. Inaccessible
types are disqualified at the outset.

_Internal Types_

Any type marked with `@internal` is removed from consideration before merging
begins. None of its fields or descriptions will factor into the final composed
type.

_Combining Descriptions_

The first non-`null` description encountered is used for the final object type's
description. If no non-`null` description is found, the resulting object type
simply has no description.

_Merging Fields_

All remaining object types contribute their fields. The algorithm gathers every
field name across these types, then calls {MergeOutputField(fields)} for each
name to reconcile any differences. If {MergeOutputField(fields)} returns {null}
(for instance, because a field is marked `@inaccessible`), that field is
excluded from the final object type. The result is a unified set of fields that
reflects each source definition while maintaining compatibility across them.

**Examples**

Here, two `Product` object types from different schemas are merged into a single
composed `Product` type.

```graphql example
# Schema A

type Product @key(fields: "id") {
  id: ID!
  name: String
}

# Schema B

type Product @key(fields: "id") {
  id: ID!
  price: Int
}

# Composed Result

type Product {
  id: ID!
  name: String
  price: Int
}
```

In this example, the `Product` type from two schemas is merged. The `id` field
is shared across both schemas, while `name` and `price` fields are contributed
by the individual source schemas. The resulting composed type includes all
fields.

Another example demonstrates preserving descriptions during merging:

```graphql example
# Schema A

"""
First Description
"""
type Order @key(fields: "id") {
  id: ID!
}

# Schema B

"""
Second Description
"""
type Order @key(fields: "id") {
  id: ID!
  total: Float
}

# Composed Result

"""
First Description
"""
type Order {
  id: ID!
  total: Float
}
```

In this case, the description from the first schema is retained, while the
fields are merged from both schemas to create the final `Order` type.

In the following example, one of the `Product` types is marked with `@internal`.
All its fields are excluded from the composed type.

```graphql example
# Schema A

type Product @key(fields: "id") {
  id: ID!
  name: String
}

# Schema B

type Product @key(fields: "id") @internal {
  id: ID!
  price: Int
}

# Composed Result

type Product {
  id: ID!
  name: String
}
```

#### Merge Output Field

**Formal Specification**

MergeOutputField(fields):

- If any {field} in {fields} is marked with `@inaccessible`
  - Return {null}
- Filter out all fields marked with `@internal` from {fields}.
- If {fields} is empty:
  - Return {null}
- Let {firstField} be the first field in {fields}.
- Let {fieldName} be the name of {firstField}.
- Let {fieldType} be the type of {firstField}.
- Let {description} be the description of {firstField}.
- For each {field} in {fields}:
  - Set {fieldType} to be the result of {LeastRestrictiveType(fieldType, type)}.
  - If {description} is {null}:
    - Let {description} be the description of {field}.
- Let {arguments} be an empty set.
- Let {argumentNames} be the set of all argument names in {fields}.
- For each {argumentName} in {argumentNames}:
  - Let {arguments} be the set of arguments with the name {argumentName} in
    {fields}.
  - Let {mergedArgument} be the result of {MergeArgumentDefinitions(arguments)}.
  - If {mergedArguments} is not {null}:
    - Add {mergedArgument} to {arguments}.
- Return a new field with the name of {fieldName}, type of {fieldType},
  arguments of {arguments}, and description of {description}.

**Explanatory Text**

The {MergeOutputField(fields)} algorithm is used when multiple fields across
different object or interface types share the same field name and must be merged
into a single composed field. This algorithm ensures that the final composed
schema has one definitive definition for that field, resolving differences in
type, description, and arguments.

_Inaccessible Fields_

If any of the fields is marked with `@inaccessible`, the entire merged field is
discarded by returning `null`. A field that cannot be exposed in a composed
schema prevents the field from being composed at all.

_Internal Fields_

Any field marked with `@internal` is removed from consideration before merging
begins. This ensures that internal fields do not appear in the final composed
schema and also do not affect the merging process. Internal fields are intended
for internal use only and are not part of the composed schema and can collide in
their definitions.

In the case where all fields are marked with `@internal`, the field will not
appear in the composed schema.

_Combining Descriptions_

The first field that defines a description is used as the description for the
merged field. If no description is found, the merged field will have no
description.

_Determining the Field Type_

The return type of the composed field is determined by invoking
{LeastRestrictiveType(typeA, typeB)}. This helper function computes a type that
is compatible with all the provided field types, ensuring that the composed
schema does not break schemas expecting any of those types. For example,
{LeastRestrictiveType(typeA, typeB)} might unify `String!` and `String` into
`String`.

_Merging Arguments_

Each field can declare arguments. The algorithm collects all all argument names
across these fields and merges them using {MergeArgumentDefinitions(arguments)},
ensuring argument definitions remain compatible. If any of the arguments for a
particular name is `@inaccessible`, then that argument is removed from the final
set of arguments. Otherwise, any differences in argument type, default value, or
description are resolved via the merging rules in
{MergeArgumentDefinitions(arguments)}.

This algorithm preserves as much information as possible from the source fields
while ensuring they remain mutually compatible. It also systematically excludes
fields or arguments deemed inaccessible.

**Example**

Imagine two schemas with a `discountPercentage` field on a `Product` type that
slightly differ in return type:

```graphql example
# Schema A

type Product {
  """
  Computes a discount as a percentage of the product's list price.
  """
  discountPercentage(percent: Int = 10): Int!
}

# Schema B

type Product {
  discountPercentage(percent: Int): Int
}

# Composed Result

type Product {
  """
  Computes a discount as a percentage of the product's list price.
  """
  discountPercentage(percent: Int): Int
}
```

#### Merge Input Field

**Formal Specification**

MergeInputField(fields):

- If any {field} in {fields} is marked with `@inaccessible`
  - Return null
- Let {firstField} be the first field in {fields}.
- Let {fieldName} be the name of {firstField}.
- Let {fieldType} be the type of {firstField}.
- Let {description} be the description of {firstField}.
- Let {defaultValue} be the default value of {firstField} or undefined if none
  exists.
- For each {field} in {fields}:
  - Set {fieldType} to be the result of {MostRestrictiveType(fieldType, type)}.
  - If {defaultValue} is undefined:
    - Set {defaultValue} to the default value of {field} or undefined if none
      exists.
  - If {description} is null:
    - Let {description} be the description of {field}.
- Return a new input field with the name of {fieldName}, type of {fieldType},
  and description of {description} and default value of {defaultValue}.

**Explanatory Text**

The {MergeInputField(fields)} algorithm merges multiple input field definitions,
all sharing the same field name, into a single composed input field. This
ensures the final input type in a composed schema maintains a consistent type,
description, and default value for that field. Below is a breakdown of how
{MergeInputField(fields)} operates:

_Inaccessible Fields_

If any of the fields is marked with `@inaccessible`, we cannot include the field
in the composed schema, and the merge algorithm returns `null`.

_Combining Descriptions_

The name of the merged field is taken from the first field in the list. The
description is set to the first non-`null` description encountered among the
fields. If no description is found, the merged field will have no description.

_Combining Field Types_

The merged field type is computed by calling {MostRestrictiveType(typeA, typeB)}
. Unlike output fields, where {LeastRestrictiveType(typeA, typeB)} is used,
input fields often follow stricter constraints. If one source schema defines a
field as non-nullable and another as nullable, the merged field type must be
non-nullable to satisfy both schemas. {MostRestrictiveType(typeA, typeB)}
ensures a final input type that is compatible with all definitions of that
field.

_Inheriting Default Values_

If multiple fields define default values, whichever appears first in the list
effectively _wins_. If there are non compatible default values, the pre merge
validation has already asserted that the default values are compatible.

**Examples**

Suppose we have two input type definitions for the same `OrderFilter` input
field, defined in separate schemas:

```graphql example
# Schema A

input OrderFilter {
  """
  Filter by the minimum order total
  """
  minTotal: Int = 0
}

# Schema B

input OrderFilter {
  minTotal: Int!
}

# Composed Result

input OrderFilter {
  """
  Filter by the minimum order total
  """
  minTotal: Int = 0
}
```

In the final schema, `minTotal` is defined using the most restrictive type
(`Int!`), has a default value of `0`, and includes the description from the
original field in `Schema A`.

#### Merge Argument Definitions

**Formal Specification**

MergeArgumentDefinitions(arguments):

- If any argument in {arguments} is marked with `@inaccessible`
  - Return null
- Let {mergedArgument} be the first argument in {arguments} that is not marked
  with `@require`
- If {mergedArgument} is null
  - Return null
- For each {argument} in {arguments}:
  - If {argument} is marked with `@require`
    - Continue
  - Set {mergedArgument} to the result of {MergeArgument(mergedArgument,
    argument)}
- Return {mergedArgument}

**Explanatory Text**

{MergeArgumentDefinitions(arguments)} merges multiple arguments that share the
same name across different field definitions into a single composed argument
definition.

_Inaccessible Arguments_

If any argument in the set is marked with `@inaccessible`, the entire argument
definition is discarded by returning `null`. An inaccessible argument should not
appear in the final composed schema.

_Handling `@require`_

The `@require` directive is used to indicate that the argument is required for
the field to be resolved, yet it specifies it as a dependency that is resolved
at runtime. Therefore, this argument should not affect the merge process. If
there are only `@require` arguments in the set, the merge algorithm returns
`null`.

_Merging Arguments_

All arguments that are not marked with `@require` are merged using the
`MergeArgument` algorithm. This algorithm ensures that the final composed
argument is compatible with all definitions of that argument, resolving
differences in type, default value, and description.

By selectively including or excluding certain arguments (via `@inaccessible` or
`@require`), and merging differences where possible, this algorithm ensures that
the resulting composed argument is both valid and compatible with the source
definitions.

**Example**

Consider two field definitions that share the same `filter` argument, but with
slightly different types and descriptions:

```graphql example
# Schema A

type Query {
  searchProducts(
    """
    Filter to apply to the search
    """
    filter: ProductFilter!
  ): [Product]
}

# Schema B

type Query {
  searchProducts(
    """
    Search filter to apply
    """
    filter: ProductFilter
  ): [Product]
}

# Composed Result

type Query {
  searchProducts(
    """
    Filter to apply to the search
    """
    filter: ProductFilter!
  ): [Product]
}
```

In the merged schema, the `filter` argument is defined with the most restrictive
type (`ProductFilter!`), includes the description from the original field in
`Schema A`, and is marked as required.

#### Merge Argument

**Formal Specification**

MergeArgument(argumentA, argumentB):

- Let {typeA} be the type of {argumentA}.
- Let {typeB} be the type of {argumentB}.
- Let {type} be {MostRestrictiveType(typeA, typeB)}.
- Let {description} be the description of {argumentA} or undefined if none
  exists.
- If {description} is undefined:
  - Let {description} be the description of {argumentB}.
- Let {defaultValue} be the default value of {argumentA} or undefined if none
  exists.
- If {defaultValue} is undefined:
  - Set {defaultValue} to the default value of {argumentB} or undefined if none
    exists.
- Return a new argument with the name of {argumentA}, type of {type}, and
  description of {description}.

**Explanatory Text**

{MergeArgument(argumentA, argumentB)} takes two arguments with the same name but
possibly differing in type, description, or default value, and returns a single,
unified argument definition.

_Unifying the Type_

The algorithm uses {MostRestrictiveType(typeA, typeB)} to determine the final
argument type. For input positions (like arguments), the most restrictive type
is needed to ensure that the merged argument type accepts all values the sources
demand. For instance, if one argument type is `String!` and the other is
`String`, the merged type must be `String!` so that it remains valid from both
perspectives.

_Choosing the Description_

The description of the first argument is used if it is defined, otherwise the
description of the second argument is used.

_Inheriting the Default Value_

The algorithm takes the first defined default value it encounters. Pre-merge
validation has already asserted that any differing defaults are compatible.

**Examples**

Suppose we have two variants of the same argument, `limit`, from different
services:

**Service A**

```graphql example
# Schema A

limit: Int = 10

# Schema B

"""
Number of items to fetch
"""
limit: Int!

# Composed Result

"""
Number of items to fetch
"""
limit: Int! = 10
```

#### Least Restrictive Type

**Formal Specification**

LeastRestrictiveType(typeA, typeB):

- Let {isNullable} be true.
- If {typeA} and {typeB} are non nullable types:
  - Set {isNullable} to false.
- If {typeA} is a non nullable type:
  - Set {typeA} to the inner type of {typeA}.
- If {typeB} is a non nullable type:
  - Set {typeB} to the inner type of {typeB}.
- If {typeA} is a list type:
  - Assert: {typeB} is a list type.
  - Let {innerTypeA} be the inner type of {typeA}.
  - Let {innerTypeB} be the inner type of {typeB}.
  - Let {innerType} be {LeastRestrictiveType(innerTypeA, innerTypeB)}.
  - If {isNullable} is true:
    - Return {innerType} as a nullable list type.
  - Otherwise:
    - Return {innerType} as a non nullable list type.
- Otherwise:
  - Assert: {typeA} is equal to {typeB}
  - If {isNullable} is true:
    - Return {typeA} as a nullable type.
  - Otherwise:
    - Return {typeA} as a non nullable type.

**Explanatory Text**

{LeastRestrictiveType(typeA, typeB)} identifies a single type that safely
handles all possible _runtime values_ produced by the sources defining `typeA`
and `typeB`. If one source can return `null` while another cannot, the merged
type becomes nullable to avoid runtime exceptions - because a strictly non-null
signature would be violated whenever `null` appears. Similarly, if both sources
enforce non-null, the result remains non-null.

_Nullability_

When merging types of differing nullability (e.g., one `String!` vs. another
`String`), the presence of a nullable type in one source effectively dictates
that the final type must accept `null`. If either source can produce `null`, a
strictly non-null field would break the contract if `null` were ever returned.

_Lists_

If both sources provide a list type, then the function unifies those list types
by merging their inner types (e.g., the element type of the list). Whether the
list itself is nullable depends on whether both sources treat the list as
non-null. In other words, if any source can return `null` for the list, the
final list type must also be nullable.

_Scalar Types_

When neither source specifies a list type, the algorithm confirms that both
sources refer to the _same_ underlying named type (e.g., `String` vs. `String`).
If they differ (e.g., `String` vs. `Int`), the schemas are fundamentally
incompatible for merging, yet the pre merge validation should have already
caught this issue.

**Examples**

In the following scenario, one source might return `null`, so the resulting
merged type must allow `null`.

```graphql example
# Schema A
typeA: String!

# Schema B
typeB: String

# Merged Result
type: String
```

Here, both sources use lists of `Int`, but they differ in nullability.
Consequently, the merged list type is `[Int]`, which permits a `null` list or
`null` elements.

```graphql example
# Schema A
typeA: [Int]!

# Schema B
typeB: [Int!]

# Merged Result
type: [Int]
```

#### Most Restrictive Type

**Formal Specification**

MostRestrictiveType(typeA, typeB):

- Let {isNullable} be false.
- If {typeA} and {typeB} are nullable types:
  - Set {isNullable} to true.
- If {typeA} is a non nullable type:
  - Set {typeA} to the inner type of {typeA}.
- If {typeB} is a non nullable type:
  - Set {typeB} to the inner type of {typeB}.
- If {typeA} is a list type:
  - Assert: {typeB} is a list type.
  - Let {innerTypeA} be the inner type of {typeA}.
  - Let {innerTypeB} be the inner type of {typeB}.
  - Let {innerType} be {MostRestrictiveType(innerTypeA, innerTypeB)}.
  - If {isNullable} is true:
    - Return {innerType} as a nullable list type.
  - Otherwise:
    - Return {innerType} as a non nullable list type.
- Otherwise
  - Assert: {typeA} is equal to {typeB}
  - If {isNullable} is true:
    - Return {typeA} as a nullable type.
  - Otherwise:
    - Return {typeA} as a non nullable type.

**Explanatory Text**

{MostRestrictiveType(typeA, typeB)} determines a single input type that strictly
honors the constraints of both sources. If either source requires a non-null
value, the merged type also becomes non-null so that no invalid (e.g., `null`)
data can be introduced at runtime. Conversely, if both sources allow `null`, the
merged type remains nullable. The same principle applies to list types, where
the more restrictive settings (non-null list or non-null elements) is used.

_Nullability_

For input fields, if either source are non null, it's unsafe to allow `null` in
the merged schema. Consequently, when one type is non-nullable (`String!`) and
the other is nullable (`String`), the resulting type is non-nullable
(`String!`). Only if _both_ types are explicitly nullable does the merged type
remain nullable (e.g., `String`).

_Lists_

When merging list types, both sources must be lists. Inside the list, the same
merging logic applies: if either source disallows `null` elements (e.g.,
`[Int!]` vs. `[Int]`), the final merged list also disallows `null` elements to
avoid unexpected runtime failures. If both lists can have `null` elements, then
the merged list similarly allows `null`.

_Scalar Types_

Like other merging steps, if the underlying base types (e.g., `String` vs.
`Int`) differ, the types cannot be reconciled. A merged schema cannot
reinterpret `String` as `Int`, so the process fails if there's a fundamental
mismatch. This should already be caught by the pre merge validation.

**Examples**

Here, because one source disallows `null`, the final merged type must also
disallow `null` to avoid a situation where a `null` could be passed where it
isn't allowed:

```graphql example
# Schema A
typeA: String!

# Schema B
typeB: String

# Merged Result
type: String!
```

In the following example, since one definition mandates non-null items
(`[Int!]`), it is _more restrictive_ and prevents `null` elements in the list.
Additionally, the other source mandates a non-null list (`[Int]!`). The merged
result, `[Int!]!`, preserves these constraints to ensure the field does not
accept or produce values that violate either source.

```graphql example
# Schema A
typeA: [Int!]

# Schema B
typeB: [Int]!

# Merged Result
type: [Int!]!
```

### Post Merge Validation

After the schema is composed, there are certain validations that are only
possible in the context of the fully merged schema. These validations verify
overall consistency: for example, ensuring that no type is left without
accessible fields, or that interfaces and their implementors remain compatible.
This stage confirms that the combined schema remains coherent when considered as
a whole.

#### Empty Merged Object Type

**Error Code**

`EMPTY_MERGED_OBJECT_TYPE`

**Severity** ERROR

**Formal Specification**

- Let {types} be the set of all input object types in the composite schema.
- For each {type} in {types}:
  - Let {fields} be a set of all fields in {type}.
  - {fields} must not be empty.

**Explanatory Text**

For object types defined across multiple source schemas, the merged object type
is the superset of all fields defined in these source schemas. However, any
field marked with `@inaccessible` in any source schema is hidden and not
included in the merged object type. An object type with no fields, after
considering `@inaccessible` annotations, is considered empty and invalid.

**Examples**

In the following example, the merged object type `Author` is valid. It includes
all fields from both source schemas, with `age` being hidden due to the
`@inaccessible` directive in one of the source schemas:

```graphql
# Schema A

type Author {
  name: String
  age: Int @inaccessible
}

# Schema B
type Author {
  age: Int
  registered: Boolean
}
```

If the `@inaccessible` directive is applied to an object type itself, the entire
merged object type is excluded from the composite execution schema, and it is
not required to contain any fields.

```graphql
# Schema A

type Author @inaccessible {
  name: String
  age: Int
}

# Schema B
type Author {
  registered: Boolean
}
```

This counter-example demonstrates an invalid merged object type. In this case,
`Author` is defined in two source schemas, but all fields are marked as
`@inaccessible` in at least one of the source schemas, resulting in an empty
merged object type:

```graphql counter-example
# Schema A

type Author {
  name: String @inaccessible
  registered: Boolean
}

# Schema B

type Author {
  name: String
  registered: Boolean @inaccessible
}
```

#### No Queries

**Error Code**

`NO_QUERIES`

**Severity**

ERROR

**Formal Specification**

- Let {fields} be the set of all fields in the `Query` type of the merged
  schema.
- {HasPublicField(fields)} must be true.

HasPublicField(fields):

- For each {field} in {fields}:
  - If {IsExposed(field)} is true
    - return true
- return false

**Explanatory Text**

This rule ensures that the composed schema includes at least one accessible
field on the root `Query` type.

In GraphQL, the `Query` type is essential as it defines the entry points for
read operations. If none of the composed schemas expose any query fields, the
composed schema would lack a root query, making it a invalid GraphQL schema.

**Examples**

In this example, at least one schema provides accessible query fields,
satisfying the rule.

```graphql
# Schema A
type Query {
  product(id: ID!): Product
}

type Product {
  id: ID!
}
```

```graphql
type Query {
  review(id: ID!): Review
}

# Schema B
type Review {
  id: ID!
  content: String
  rating: Int
}
```

Even if some query fields are marked as `@inaccessible`, as long as there is at
least one accessible query field in the composed schema, the rule is satisfied.

In this case, Schema A exposes an internal query field `internalData` marked
with `@inaccessible`, making it hidden in the composed schema. However, Schema B
provides an accessible `product` query field. Therefore, the composed schema has
at least one accessible query field, adhering to the rule.

```graphql
# Schema A
type Query {
  internalData: InternalData @inaccessible
}

type InternalData {
  secret: String
}
```

```graphql
# Schema B
type Query {
  product(id: ID!): Product
}

type Product {
  id: ID!
  name: String
}
```

If all query fields in all schemas are marked as `@inaccessible`, the composed
schema will lack accessible query fields, violating the rule.

In the following counter-example, both schemas have query fields, but all are
marked as `@inaccessible`.

This means there are no accessible query fields in the composed schema,
triggering the `NO_QUERIES` error.

```graphql
# Schema A
type Query {
  internalData: InternalData @inaccessible
}

type InternalData {
  secret: String
}
```

```graphql
# Schema B
type Query {
  adminStats: AdminStats @inaccessible
}

type AdminStats {
  userCount: Int
}
```

#### Implemented by Inaccessible

**Error Code**

`IMPLEMENTED_BY_INACCESSIBLE`

**Severity**

ERROR

**Formal Specification**

- Let {schema} be the merged composite execution schema.
- Let {types} be the set of all object types in {schema}.
- For each {type} in {types}:
  - If {type} is not marked with `@inaccessible`:
    - Let {implementedInterfaces} be the set of all interfaces implemented by
      {type}.
    - For each {field} in {type}'s fields:
      - If {field} is marked with `@inaccessible`:
        - For each {implementedInterface} in {implementedInterfaces}:
          - Let {interfaceField} be the field on {implementedInterface} that has
            the same name as {field}
          - If {interfaceField} exists:
            - {IsExposed(interfaceField)} must be false

**Explanatory Text**

This rule ensures that inaccessible fields (`@inaccessible`) on an object type
are not exposed through an interface. An object type that implements an
interface must provide public access to each field defined by the interface. If
a field on an object type is marked as `@inaccessible` but implements an
interface field that is visible in the composed schema, this creates a
contradiction: the interface contract requires that field to be accessible, yet
the object type implementation hides it.

This rule prevents inconsistencies in the composed schema, ensuring that every
interface field visible in the composed schema is also publicly visible on all
types implementing that interface.

**Examples**

In the following example, `User.id` is accessible and implements `Node.id` which
is also accessible, no error occurs.

```graphql
# The interface field `id` is visible and provided by `User` without @inaccessible.
interface Node {
  id: ID!
}

type User implements Node {
  id: ID!
  name: String
}
```

Since `Auditable` and its field `lastAudit` are `@inaccessible`, the
`Order.lastAudit` field is allowed to be `@inaccessible` because it does not
implement any visible interface field in the composed schema.

```graphql
# The entire interface is @inaccessible, thus its fields are not publicly visible.
interface Auditable @inaccessible {
  lastAudit: DateTime!
}

type Order implements Auditable {
  lastAudit: DateTime! @inaccessible
  orderNumber: String
}
```

In this example, `Node.id` is visible in the public schema (no `@inaccessible`),
but `User.id` is marked `@inaccessible`. This violates the interface contract
because `User` claims to implement `Node`, yet does not expose the `id` field to
the public schema.

```graphql counter-example
interface Node {
  id: ID!
}

type User implements Node {
  id: ID! @inaccessible
  name: String
}
```

#### Interface Field No Implementation

**Error Code**

`INTERFACE_FIELD_NO_IMPLEMENTATION`

**Severity**

ERROR

**Formal Specification**

- Let {schema} be the merged composite execution schema.
- Let {objectTypes} be the set of all object types defined in {schema}.
- For each {objectType} in {objectTypes}:
  - Let {interfaces} be the set of interface types that {objectType} implements.
  - For each {interface} in {interfaces}:
    - Let {interfaceFields} be the set of fields defined on {interface} that are
      visible in the merged schema.
    - For each {field} in {interfaceFields}:
      - If {field} is not present on {objectType}:
        - Produce an `INTERFACE_FIELD_NO_IMPLEMENTATION` error.

**Explanatory Text**

In GraphQL, any object type that implements an interface must provide a field
definition for every field declared by that interface. If an object type fails
to implement a particular field required by one of its interfaces, the composite
schema becomes invalid because the resulting schema breaks the contract defined
by that interface.

This rule checks that object types merged from different sources correctly
implement all interface fields. In scenarios where a schema defines an interface
field, but the implementing object type in another schema omits that field, an
error is raised.

**Examples**

In this valid example, the `User` interface has three fields: `id`, `name`, and
`email`. Both the `RegisteredUser` and `GuestUser` types implement all three
fields, satisfying the interface contract.

```graphql example
# Schema A
interface User {
  id: ID!
  name: String!
  email: String
}

type RegisteredUser implements User {
  id: ID!
  name: String!
  email: String
  lastLogin: DateTime
}

# Schema B
interface User {
  id: ID!
  name: String!
  email: String
}

type GuestUser implements User {
  id: ID!
  name: String!
  email: String
  temporaryCartId: String
}
```

In this counter-example, the `User` interface is defined with three fields, but
the `GuestUser` type omits one of them (`email`), causing an
`INTERFACE_FIELD_NO_IMPLEMENTATION` error.

Although `GuestUser` implements `User`, it does not provide the `email` field.
Since the merged schema sees that the interface `User` has `email` but
`GuestUser` does not provide it, the schema composition fails with the
`INTERFACE_FIELD_NO_IMPLEMENTATION` error.

```graphql counter-example
# Schema A
interface User {
  id: ID!
  name: String!
  email: String
}

type RegisteredUser implements User {
  id: ID!
  name: String!
  email: String
  lastLogin: DateTime
}

# Schema B
interface User {
  id: ID!
  name: String!
}

type GuestUser implements User {
  id: ID!
  name: String!
  temporaryCartId: String
}
```

#### Invalid Field Sharing

**Error Code**

`INVALID_FIELD_SHARING`

**Severity**

ERROR

**Formal Specification**

- Let {schema} be the merged composite execution schema.
- Let {types} be the set of all object and interface types in {schema}.
- For each {type} in {types}:
  - If {type} is the `Subscription` type:
    - Let {fields} be the set of all fields in {type}.
    - For each {field} in {fields}:
      - If {field} is marked with `@shareable`:
        - Produce an `INVALID_FIELD_SHARING` error.
  - Otherwise:
    - Let {fields} be the set of all fields on {type}.
    - For each {field} in {fields}:
      - If {field} is not part of a `@key` directive:
        - Let {fieldDefinitions} be the set of all field definitions for {field}
          across all source schemas excluding fields marked with `@external` or
          `@override`.
        - If {fieldDefinitions} has more than one element:
          - {field} must be marked as `@shareable` in at least one schema.

**Explanatory Text**

A field in a federated GraphQL schema may be marked `@shareable`, indicating
that the same field can be resolved by multiple schemas without conflict. When a
field is **not** marked as `@shareable` (sometimes called "non-shareable"), it
cannot be provided by more than one schema.

Field definitions marked as `@external` or `@override` are excluded when
validating whether a field is shareable. These annotations indicate specific
cases where field ownership lies with another schema or has been replaced.

Additionally, subscription root fields cannot be shared (i.e., they are
effectively non-shareable), as subscription events from multiple schemas would
create conflicts in the composed schema. Attempting to mark a subscription field
as shareable or to define it in multiple schemas triggers the same error.

**Examples**

In this example, the `User` type field `fullName` is marked as shareable in both
schemas, allowing them to serve consistent data for that field without conflict.

```graphql example
# Schema A
type User @key(fields: "id") {
  id: ID!
  username: String
  fullName: String @shareable
}

# Schema B
type User @key(fields: "id") {
  id: ID!
  fullName: String @shareable
  email: String
}
```

In the following example, `User.fullName` is overridden in one schema and
therefore the field can be define in multiple schemas without being marked as
`@shareable`.

```graphql example
# Schema A
type User @key(fields: "id") {
  id: ID!
  fullName: String @override(from": "B")
}

# Schema B
type User @key(fields: "id") {
  id: ID!
  fullName: String
}
```

In the following example, `User.fullName` is marked as `@external` in one schema
and therefore the field can be define in the other schema without being marked
as `@shareable`.

```graphql example
# Schema A
type User @key(fields: "id") {
  id: ID!
  fullName: String @external
}

# Schema B
type User @key(fields: "id") {
  id: ID!
  fullName: String
}
```

In the following counter-example, `User.profile` is non-shareable but is defined
and resolved by two different schemas, resulting in an `INVALID_FIELD_SHARING`
error.

```graphql counter-example
# Schema A
type User @key(fields: "id") {
  id: ID!
  profile: Profile
}

type Profile {
  avatarUrl: String
}

# Schema B
type User @key(fields: "id") {
  id: ID!
  profile: Profile
}

type Profile {
  avatarUrl: String
}
```

By definition, root subscription fields cannot be shared across multiple
schemas. In this example, both schemas define a subscription field
`newOrderPlaced`:

```graphql counter-example
# Schema A
type Subscription {
  newOrderPlaced: Order @shareable
}

type Order {
  id: ID!
  items: [String]
}

# Schema B
type Subscription {
  newOrderPlaced: Order @shareable
}
```

#### Invalid Shareable Usage

**Error Code**

`INVALID_SHAREABLE_USAGE`

**Severity**

ERROR

**Formal Specification**

- Let {schema} be one of the composed schemas.
- Let {types} be the set of types defined in {schema}.
- For each {type} in {types}:
  - If {type} is an interface type:
    - For each field definition {field} in {type}:
      - If {field} is annotated with `@shareable`, produce an
        `INVALID_SHAREABLE_USAGE` error.

**Explanatory Text**

The `@shareable` directive is intended to indicate that a field on an **object
type** can be resolved by multiple schemas without conflict. As a result, it is
only valid to use `@shareable` on fields **of object types** (or on the entire
object type itself).

Applying `@shareable` to interface fields is disallowed and violates the valid
usage of the directive. This rule prevents schema composition errors and data
conflicts by ensuring that `@shareable` is used only in contexts where shared
field resolution is meaningful and unambiguous.

**Examples**

In this example, the field `orderStatus` on the `Order` object type is marked
with `@shareable`, which is allowed. It signals that this field can be served
from multiple schemas without creating a conflict.

```graphql example
type Order {
  id: ID!
  orderStatus: String @shareable
  total: Float
}
```

In this example, the `InventoryItem` interface has a field `sku` marked with
`@shareable`, which is invalid usage. Marking an interface field as shareable
leads to an `INVALID_SHAREABLE_USAGE` error.

```graphql counter-example
interface InventoryItem {
  sku: ID! @shareable
  name: String
}
```

#### Only Inaccessible Children

**Error Code**

`ONLY_INACCESSIBLE_CHILDREN`

**Severity**

ERROR

**Formal Specification**

- Let {schema} be the composed schema.
- Let {types} be the set of all types in {schema}.
- For each {type} in {types}:
  - If {IsExposed(type)} is false:
    - continue
  - If {type} is the query, mutation, or subscription root type:
    - continue
  - If {type} is an object type:
    - {HasObjectTypeAccessibleChildren(type)} must be true
  - If {type} is an enum type:
    - {HasEnumAccessibleChildren(type)} must be true
  - If {type} is an input object type:
    - {HasInputObjectAccessibleChildren(type)} must be true
  - If {type} is an interface type:
    - {HasInterfaceAccessibleChildren(type)} must be true
  - If {type} is a union type:
    - {HasUnionAccessibleChildren(type)} must be true

HasObjectTypeAccessibleChildren(type):

- Let {fields} be the set of all fields in {type}.
- For each {field} in {fields}:
  - If {field} is **not** marked with `@inaccessible` and **not** `@internal`:
    - return true
- return false

HasEnumAccessibleChildren(type):

- Let {values} be the set of all values in {type}.
- For each {value} in {values}:
  - If {value} is **not** marked with `@inaccessible`:
    - return true
- return false

HasInputObjectAccessibleChildren(type):

- Let {fields} be the set of all fields in {type}.
- For each {field} in {fields}:
  - If {value} is **not** marked with `@inaccessible`:
    - return true
- return false

HasInterfaceAccessibleChildren(type):

- Let {fields} be the set of all fields in {type}.
- For each {field} in {fields}:
  - If {field} is **not** marked with `@inaccessible`:
    - return true
- return false

HasUnionAccessibleChildren(type):

- Let {members} be the set of all member types in {type}.
- For each {member} in {members}:
  - Let {type} be the type of {member}.
  - If {type} is **not** marked with `@inaccessible`:
    - return true
- return false

**Explanatory Text**

A type that is **not** annotated with `@inaccessible` is expected to appear in
the composed schema. If, however, all of its child elements (fields in an object
or interface, values in an enum, fields in an input object or all types of a
union) are individually marked `@inaccessible` (or `@internal`), then there are
no accessible sub-parts of that type for consumers to query or reference.

Allowing such a type to remain in the composed schema despite having no publicly
visible fields or values leads to an invalid schema. This rule enforces that a
type visible in the composed schema must have at least one accessible child.
Otherwise, it triggers an `ONLY_INACCESSIBLE_CHILDREN` error, prompting the user
to either make the entire type `@inaccessible` or expose at least one child
element.

Additionally, the rule applies to all types except the query, mutation, and
subscription root types.

**Examples**

In the following example, the `Profile` type is included in the composed schema,
and `Profile.email` is **not** marked with `@inaccessible`. This satisfies the
rule, as there is at least one accessible child element.

```graphql
type User {
  id: ID!
  profile: Profile
}

type Profile {
  name: String @inaccessible
  email: String
}
```

In the following example, all fields of the `Profile` type are marked with
`@inaccessible`. But since `Profile` itself is marked with `@inaccessible`, it
is not required to have any accessible children.

```graphql
type User {
  id: ID!
  profile: Profile @inaccessible
}

type Profile @inaccessible {
  name: String @inaccessible
  email: String @inaccessible
}
```

The `Profile` type is included in the composed schema (no `@inaccessible` on the
type), but **all** of its fields are marked `@inaccessible`, triggering an
`ONLY_INACCESSIBLE_CHILDREN` error.

```graphql counter-example
type User {
  id: ID!
  profile: Profile
}

type Profile {
  name: String @inaccessible
  email: String @inaccessible
}
```

In this example, the `DeliveryStatus` enum is not annotated with
`@inaccessible`, yet all of its values are.

Since there are no publicly visible values, an `ONLY_INACCESSIBLE_CHILDREN`
error is produced.

```graphql counter-example
enum DeliveryStatus {
  PENDING @inaccessible
  SHIPPED @inaccessible
  DELIVERED @inaccessible
}
```

#### Require Invalid Fields

**Error Code**

`REQUIRE_INVALID_FIELDS`

**Severity**

ERROR

**Formal Specification**

- Let {schema} be the merged composite execution schema.
- Let {compositeTypes} be the set of all composite types in {schema}.
- For each {composite} in {compositeTypes}:
  - Let {fields} be the set of fields on {composite}.
  - Let {arguments} be the set of all arguments on {fields}.
  - For each {argument} in {arguments}:
    - If {argument} is **not** annotated with `@require`:
      - Continue
    - Let {fieldsArg} be the string value of the `fields` argument of the
      `@require` directive on {argument}.
    - Let {parsedFieldsArg} be the parsed selection map from {fieldsArg}.
    - {ValidateSelectionMap(parsedFieldsArg, parentType)} must be true.

ValidateSelectionMap(selectionMap, parentType):

- For each {selection} in {selectionMap}:
  - Let {field} be the field selected by {selection} on {parentType}.
  - If {field} is **not** defined on {parentType}:
    - return false
  - Let {fieldType} be the type of {field}.
  - If {fieldType} is not a scalar type
    - Let {subSelections} be the selections in {selection}
    - If {subSelections} is empty
      - return false
    - If {ValidateSelectionMap(subSelections, fieldType)} is false
      - return false
- return true

**Explanatory Text**

Even if the selection map for `@require(fields: "…")` is syntactically valid,
its contents must also be valid within the composed schema. Fields must exist on
the parent type for them to be referenced by `@require`. In addition, fields
requiring unknown fields break the valid usage of `@require`, leading to a
`REQUIRE_INVALID_FIELDS` error.

**Examples**

In the following example, the `@require` directive's `fields` argument is a
valid selection set and satisfies the rule.

```graphql example
type User @key(fields: "id") {
  id: ID!
  name: String!
  profile(name: String! @require(fields: "name")): Profile
}

type Profile {
  id: ID!
  name: String
}
```

In this counter-example, the `@require` directive does not have a valid
selection set and triggers a `REQUIRE_INVALID_FIELDS` error.

```graphql counter-example
type Book {
  id: ID!
  title(lang: String! @require(fields: "author { }")): String
}

type Author {
  name: String
}
```

In this counter-example, the `@require` directive references a field (`unknown`)
that does not exist on the parent type (`Book`), causing a
`REQUIRE_INVALID_FIELDS` error.

```graphql counter-example
type Book {
  id: ID!
  pages(pageSize: Int @require(fields: "unknownField")): Int
}
```

#### Provides Invalid Fields

**Error Code**

`PROVIDES_INVALID_FIELDS`

**Severity**

ERROR

**Formal Specification**

- Let {schema} be the merged composite execution schema.
- Let {fieldsWithProvides} be the set of all fields annotated with the
  `@provides` directive in {schema}.
- For each {field} in {fieldsWithProvides}:
  - Let {fieldsArg} be the string value of the `fields` argument of the
    `@provides` directive on {field}.
  - Let {parsedSelectionSet} be the parsed selection set from {fieldsArg}.
  - Let {returnType} be the return type of {field}.
  - {ValidateSelectionSet(parsedSelectionSet, returnType)} must be true.

ValidateSelectionSet(selectionSet, parentType):

- For each {selection} in {selectionSet}:
  - Let {selectedField} be the field named by {selection} in {parentType}.
  - If {selectedField} does not exist on {parentType}:
    - return false
  - If {selectedField} returns a composite type then {selection}
    - Let {subSelections} be the selections in {selection}
    - If {subSelections} is empty
      - return false
    - If {ValidateSelectionSet(subSelections, fieldType)} is false
      - return false
- return true

**Explanatory Text**

Even if the `@provides(fields: "…")` argument is well-formed syntactically, the
selected fields must actually exist on the return type of the field. Invalid
field references- e.g., selecting non-existent fields, referencing fields on the
wrong type, or incorrectly omitting required nested selections-lead to a
`PROVIDES_INVALID_FIELDS` error.

**Examples**

In the following example, the `@provides` directive references a valid field
(`hobbies`) on the `UserDetails` type.

```graphql example
type User @key(fields: "id") {
  id: ID!
  details: UserDetails @provides(fields: "hobbies")
}

type UserDetails {
  hobbies: [String]
}
```

In the following counter-example, the `@provides` directive specifies a field
named `unknownField` which is not defined on `UserDetails`. This raises a
`PROVIDES_INVALID_FIELDS` error.

```graphql counter-example
type User @key(fields: "id") {
  id: ID!
  details: UserDetails @provides(fields: "unknownField")
}

type UserDetails {
  hobbies: [String]
}
```

#### Empty Merged Input Object Type

**Error Code**

`EMPTY_MERGED_INPUT_OBJECT_TYPE`

**Severity**

ERROR

**Formal Specification**

- Let {inputTypes} be the set of all input object types in the composite schema.
- For each {inputType} in {inputTypes}:
  - Let {fields} be a set of all fields in {inputType}.
  - {fields} must not be empty.

**Explanatory Text**

For input object types defined across multiple source schemas, the merged input
object type is the intersection of all fields defined in these source schemas.
Any field marked with the `@inaccessible` directive in any source schema is
hidden and not included in the merged input object type. An input object type
with no fields, after considering `@inaccessible` annotations, is considered
empty and invalid.

**Examples**

In the following example, the merged input object type `BookFilter` is valid.

```graphql
input BookFilter {
  name: String
}

input BookFilter {
  name: String
}
```

If the `@inaccessible` directive is applied to an input object type itself, the
entire merged input object type is excluded from the composite execution schema,
and it is not required to contain any fields.

```graphql
input BookFilter @inaccessible {
  name: String
  minPageCount: Int
}

input BookFilter {
  name: Boolean
}
```

This counter-example demonstrates an invalid merged input object type. In this
case, `BookFilter` is defined in two source schemas, but all fields are marked
as `@inaccessible` in at least one of the source schemas, resulting in an empty
merged input object type:

```graphql counter-example
input BookFilter {
  name: String @inaccessible
  paperback: Boolean
}

input BookFilter {
  name: String
  paperback: Boolean @inaccessible
}
```

Here is another counter-example where the merged input object type is empty
because no fields intersect between the two source schemas:

```graphql counter-example
input BookFilter {
  paperback: Boolean
}

input BookFilter {
  name: String
}
```

#### Non-Null Input Fields cannot be inaccessible

**Error Code**

`NON_NULL_INPUT_FIELD_IS_INACCESSIBLE`

**Formal Specification**

- Let {fields} be the set of all fields across all input types in all source
  schemas.
- For each {field} in {fields}:
  - If {field} is a non-null input field:
    - Let {coordinate} be the coordinate of {field}.
    - {coordinate} must be in the composite schema.

**Explanatory Text**

When an input field is declared as non-null in any source schema, it imposes a
hard requirement: queries or mutations that reference this field _must_ provide
a value for it. If the field is then marked as `@inaccessible` or removed during
schema composition, the final schema would still implicitly demand a value for a
field that no longer exists in the composed schema, making it impossible to
fulfill the requirement.

As a result:

- **Nullable** (optional) fields can be hidden or removed without invalidating
  the composed schema, because the user is never _required_ to supply a value
  for them.
- **Non-null** (required) fields, however, must remain exposed in the composed
  schema so that users can provide values for those fields. Hiding a required
  input field breaks the schema contract and leads to an invalid composition.

**Examples**

The following is valid because the `age` field, although `@inaccessible` in one
source schema, is nullable and can be safely omitted in the final schema without
breaking any mandatory input requirement.

```graphql example
# Schema A
input BookFilter {
  author: String!
  age: Int @inaccessible
}

# Schema B
input BookFilter {
  author: String!
  age: Int
}

# Composite Schema
input BookFilter {
  author: String!
}
```

Another valid case is when a nullable input field is removed during merging:

```graphql example
# Schema A
input BookFilter {
  author: String!
  age: Int
}

# Schema B
input BookFilter {
  author: String!
}

# Composite Schema
input BookFilter {
  author: String!
}
```

An invalid case is when a non-null input field is inaccessible:

```graphql counter-example
# Schema A
input BookFilter {
  author: String!
  age: Int!
}

# Schema B
input BookFilter {
  author: String!
  age: Int @inaccessible
}

# Composite Schema
input BookFilter {
  author: String!
}
```

Another invalid case is when a non-null input field is removed during merging:

```graphql counter-example
# Schema A
input BookFilter {
  author: String!
  age: Int!
}

# Schema B
input BookFilter {
  author: String!
}

# Composite Schema
input BookFilter {
  author: String!
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

The final step confirms that the composite schema supports executable queries
without leading to invalid conditions. Each query path defined in the merged
schema is checked to ensure that every field can be resolved. If any query path
is unresolvable, the schema is deemed unsatisfiable, and composition fails.
