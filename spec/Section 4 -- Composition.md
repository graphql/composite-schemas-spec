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

### Requires Directive in Fields Argument

**Error Code**

`REQUIRES_DIRECTIVE_IN_FIELDS_ARG`

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
      - If {argument} is **not** marked with `@requires`:
        - Continue
      - Let {fieldsArg} be the value of the `fields` argument of the `@requires`
        directive on {argument}.
      - If {fieldsArg} contains a directive application:
        - Produce a `REQUIRES_DIRECTIVE_IN_FIELDS_ARG` error.

**Explanatory Text**

The `@requires` directive is used to specify fields on the same type that an
argument depends on in order to resolve the annotated field.  
When using `@requires(fields: "…")`, the `fields` argument must be a valid
selection set string **without** any additional directive applications.  
Applying a directive (e.g., `@lowercase`) inside this selection set is not
supported and triggers the `REQUIRES_DIRECTIVE_IN_FIELDS_ARG` error.

**Examples**

In this valid usage, the `@requires` directive’s `fields` argument references
`name` without any directive applications, avoiding the error.

```graphql example
type User @key(fields: "id name") {
  id: ID!
  profile(name: String! @requires(fields: "name")): Profile
}

type Profile {
  id: ID!
  name: String
}
```

Because the `@requires` selection (`name @lowercase`) includes a directive
application (`@lowercase`), this violates the rule and triggers a
`REQUIRES_DIRECTIVE_IN_FIELDS_ARG` error.

```graphql counter-example
type User @key(fields: "id name") {
  id: ID!
  name: String
  profile(name: String! @requires(fields: "name @lowercase")): Profile
}

type Profile {
  id: ID!
  name: String
}
```

### Requires Invalid Fields Type

**Error Code**

`REQUIRES_INVALID_FIELDS_TYPE`

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
      - If {argument} is **not** annotated with `@requires`:
        - Continue
      - Let {fieldsArg} be the value of the `fields` argument of the `@requires`
        directive on {argument}.
      - If {fieldsArg} is **not** a string:
        - Produce a `REQUIRES_INVALID_FIELDS_TYPE` error.

**Explanatory Text**

When using the `@requires` directive, the `fields` argument must always be a
string that defines a (potentially nested) selection set of fields from the same
type. If the `fields` argument is provided as a type other than a string (such
as an integer, boolean, or enum), the directive usage is invalid and will cause
schema composition to fail.

**Examples**

In the following example, the `@requires` directive’s `fields` argument is a
valid string and satisfies the rule.

```graphql example
type User @key(fields: "id") {
  id: ID!
  profile(name: String! @requires(fields: "name")): Profile
}

type Profile {
  id: ID!
  name: String
}
```

Since `fields` is set to `123` (an integer) instead of a string, this violates
the rule and triggers a `REQUIRES_INVALID_FIELDS_TYPE` error.

```graphql counter-example
type User @key(fields: "id") {
  id: ID!
  profile(name: String! @requires(fields: 123)): Profile
}

type Profile {
  id: ID!
  name: String
}
```

### Requires Invalid Syntax

**Error Code**

`REQUIRES_INVALID_SYNTAX`

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
      - If {argument} is **not** annotated with `@requires`:
        - Continue
      - Let {fieldsArg} be the string value of the `fields` argument of the
        `@requires` directive on {argument}.
      - {fieldsArg} must be be parsable as a valid selection map

**Explanatory Text**

The `@requires` directive’s `fields` argument must be syntactically valid
GraphQL. If the selection map string is malformed (e.g., missing closing braces,
unbalanced quotes, invalid tokens), then the schema cannot be composed
correctly. In such cases, the error `REQUIRES_INVALID_SYNTAX` is raised.

**Examples**

In the following example, the `@requires` directive’s `fields` argument is a
valid selection map and satisfies the rule.

```graphql example
type User @key(fields: "id") {
  id: ID!
  profile(name: String! @requires(fields: "name")): Profile
}

type Profile {
  id: ID!
  name: String
}
```

In the following counter-example, the `@requires` directive’s `fields` argument
has invalid syntax because it is missing a closing brace.

This violates the rule and triggers a `REQUIRES_INVALID_FIELDS` error.

```graphql counter-example
type Book {
  id: ID!
  title(lang: String! @requires(fields: "author { name ")): String
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

### Implemented by Inaccessible

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

### Interface Field No Implementation

**Error Code**

`INTERFACE_FIELD_NO_IMPLEM`

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
        - Produce an `INTERFACE_FIELD_NO_IMPLEM` error.

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
`INTERFACE_FIELD_NO_IMPLEM` error.

Although `GuestUser` implements `User`, it does not provide the `email` field.
Since the merged schema sees that the interface `User` has `email` but
`GuestUser` does not provide it, the schema composition fails with the
`INTERFACE_FIELD_NO_IMPLEM` error.

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

### Invalid Field Sharing

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

In the following example, `User.fullName` is overriden in one schema and
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

### Invalid Shareable Usage

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

### Only Inaccessible Children

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

### Requires Invalid Fields

**Error Code**

`REQUIRES_INVALID_FIELDS`

**Severity**

ERROR

**Formal Specification**

- Let {schema} be the merged composite execution schema.
- Let {compositeTypes} be the set of all composite types in {schema}.
- For each {composite} in {compositeTypes}:
  - Let {fields} be the set of fields on {composite}.
  - Let {arguments} be the set of all arguments on {fields}.
  - For each {argument} in {arguments}:
    - If {argument} is **not** annotated with `@requires`:
      - Continue
    - Let {fieldsArg} be the string value of the `fields` argument of the
      `@requires` directive on {argument}.
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

Even if the selection map for `@requires(fields: "…")` is syntactically valid,
its contents must also be valid within the composed schema. Fields must exist on
the parent type for them to be referenced by `@requires`. In addition, fields
requiring unknown fields break the valid usage of `@requires`, leading to a
`REQUIRES_INVALID_FIELDS` error.

**Examples**

In the following example, the `@requires` directive’s `fields` argument is a
valid selection set and satisfies the rule.

```graphql example
type User @key(fields: "id") {
  id: ID!
  name: String!
  profile(name: String! @requires(fields: "name")): Profile
}

type Profile {
  id: ID!
  name: String
}
```

In this counter-example, the `@requires` directive does not have a valid
selection set and triggers a `REQUIRES_INVALID_FIELDS` error.

```graphql counter-example
type Book {
  id: ID!
  title(lang: String! @requires(fields: "author { }")): String
}

type Author {
  name: String
}
```

In this counter-example, the `@requires` directive references a field
(`unknown`) that does not exist on the parent type (`Book`), causing a
`REQUIRES_INVALID_FIELDS` error.

```graphql counter-example
type Book {
  id: ID!
  pages(pageSize: Int @requires(fields: "unknownField")): Int
}
```

### Provides Invalid Fields

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
field references— e.g., selecting non-existent fields, referencing fields on the
wrong type, or incorrectly omitting required nested selections—lead to a
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

## Validate Satisfiability
