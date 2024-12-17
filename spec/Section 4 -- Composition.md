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

This rule enforces that, for any subgraph schema, if a root mutation type is
defined, it must be named `Mutation`. Defining a root mutation type with a name
other than `Mutation` or using a differently named type alongside a type
explicitly named `Mutation` creates inconsistencies in schema design and
violates the composite schema specification.

Valid Example:

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

This rule enforces that the root query type in any subgraph schema must be named
`Query`. Defining a root query type with a name other than `Query` or using a
differently named type alongside a type explicitly named `Query` creates
inconsistencies in schema design and violates the composite schema
specification.

**Examples**

Valid Example:

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
# Subgraph A
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

This rule enforces that, for any subgraph schema, if a root subscription type is
defined, it must be named `Subscription`. Defining a root subscription type with
a name other than `Subscription` or using a differently named type alongside a
type explicitly named `Subscription` creates inconsistencies in schema design
and violates the composite schema specification.

**Examples**

Valid Example:

```graphql example
# Subgraph A
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
# Subgraph A
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
  - Let {types} be the set of all object types that are annotated with the
    `@key` directive in {schema}.
  - For each {type} in {types}:
    - Let {keyFields} be the set of fields referenced by the `fields` argument
      of the `@key` directive on {type}.
    - For each {field} in {keyFields}:
      - Let {fieldType} be the type of {field}.
      - {fieldType} must not be a `List`, `Interface`, or `Union` type.

**Explanatory Text**

The `@key` directive is used to define the set of fields that uniquely identify
an entity. These fields must reference scalars or object types to ensure a valid
and consistent representation of the entity across subgraphs. Fields of types
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

In this following counter example, the `@key` directive references a field
(`tags`) of type `List`, which is also not allowed.

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
  - Let {types} be the set of all object types that are annotated with the
    `@key` directive in {schema}.
  - For each {type} in {types}:
    - Let {fields} be the string value of the `fields` argument of the `@key`
      directive on {type}.
    - For each {selection} in {fields}:
      - {selection} must not contain a directive application.

**Explanatory Text**

The `@key` directive specifies the set of fields used to uniquely identify an
entity. The `fields` argument must consist of a valid GraphQL selection set that
does not include any directive applications. Directives in the `fields` argument
are not supported.

**Examples**

In this example, the `fields` argument of the `@key` directive does not include
any directive applications, satisfying the rule.

```graphql example
directive @lowercase on FIELD_DEFINITION

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
entity resolution across subgraphs. Fields included in the `fields` argument of
the `@key` directive must be static and consistently resolvable.

**Examples**

In this example, the `User` type has a valid `@key` directive that references
the argument free fields `id` and `name`.

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

### Key Invalid Fields

**Error Code**

`KEY_INVALID_FIELDS`

**Severity**

ERROR

**Formal Specification**

- Let {schema} be the set of all source schemas.
  - Let {types} be the set of all object types that are annotated with the
    `@key` directive in {schema}.
  - For each {type} in {types}:
    - Let {fields} be the set of string values of the `fields` argument of the
      `@key` directive on {type}.
    - For each {field} in {fields}:
      - IsValidKeyField(field, type) must be true.

IsValidKeyField(field, type):

- If {field} is not defined on {type}:
  - return false
- If {field} has a selection set:
  - Let {subType} be the return type of {field}.
  - Let {subFields} be the set of all fields in the selection set of {field}.
  - For each {subField} in {subFields}:
    - IsValidKeyField(subField, subType) must be true.
- return true

**Explanatory Text**

The `@key` directive specifies the set of fields used to uniquely identify an
entity. The `fields` argument must be valid and meet the following conditions:

1. It must have valid GraphQL syntax.
2. It must reference fields that are defined on the annotated type.

Violations of these conditions result in an invalid schema composition, as the
entity key cannot be properly resolved.

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

In this counter-example, the `fields` argument of the `@key` directive has
invalid syntax because it is missing a closing brace.

```graphql counter-example
type Product @key(fields: "featuredItem { id") {
  featuredItem: Node!
  sku: String!
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
directive @lowercase on FIELD_DEFINITION

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
  - If {IsExposed(field)} is true
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
read operations. If none of the composed subgraphs expose any query fields, the
composed schema would lack a root query, making it a invalid GraphQL schema.

**Examples**

In this example, at least one subgraph provides accessible query fields,
satisfying the rule.

```graphql
# Subgraph A
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

# Subgraph B
type Review {
  id: ID!
  content: String
  rating: Int
}
```

Even if some query fields are marked as `@inaccessible`, as long as there is at
least one accessible query field in the composed schema, the rule is satisfied.

In this case, Subgraph A exposes an internal query field `internalData` marked
with `@inaccessible`, making it hidden in the composed schema. However, Subgraph
B provides an accessible `product` query field. Therefore, the composed schema
has at least one accessible query field, adhering to the rule.

```graphql
# Subgraph A
type Query {
  internalData: InternalData @inaccessible
}

type InternalData {
  secret: String
}
```

```graphql
# Subgraph B
type Query {
  product(id: ID!): Product
}

type Product {
  id: ID!
  name: String
}
```

If all query fields in all subgraphs are marked as `@inaccessible`, the composed
schema will lack accessible query fields, violating the rule.

In the following counter-example, both subgraphs have query fields, but all are
marked as `@inaccessible`.

This means there are no accessible query fields in the composed schema,
triggering the `NO_QUERIES` error.

```graphql
# Subgraph A
type Query {
  internalData: InternalData @inaccessible
}

type InternalData {
  secret: String
}
```

```graphql
# Subgraph B
type Query {
  adminStats: AdminStats @inaccessible
}

type AdminStats {
  userCount: Int
}
```

## Validate Satisfiability
