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

### Validate Type System

#### Invalid GraphQL

**Error Code**

`INVALID_GRAPHQL`

**Severity**

ERROR

**Formal Specification**

- Let {schema} be the source schema to validate.
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

#### Disallowed Inaccessible Elements

**Error Code**

`DISALLOWED_INACCESSIBLE`

**Severity**

ERROR

**Formal Specification**

- Let {type} be the set of all types the schema.
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
  fields(includeDeprecated: Boolean! = false): [__Field!]
}
```

#### Type Definition Invalid

**Error Code**

`TYPE_DEFINITION_INVALID`

**Severity**

ERROR

**Formal Specification**

- Let {types} be the set of built-in types (for example, `FieldSelectionMap`)
  defined by the composition specification from the schema.
- For each {type} in {types}:
  - Let {kind} be the kind of {type}.
  - {kind} must be equal to the kind defined by the composition specification.
  - If {type} is a directive:
    - Let {expectedArguments} be the set of arguments defined by the composition
      specification.
    - For each {expectedArgument} in {expectedArguments}:
      - Let {name} be the name of {expectedArgument}.
      - Let {argument} be the argument with {name} in {type}.
      - {argument} must be defined.
      - Let {expectedType} be the type of {expectedArgument}.
      - Let {type} be the type of {argument}.
      - {type} must be equal to {expectedType}.

**Explanatory Text**

Certain types (and directives) are reserved in composite schema specification
for specific purposes and must adhere to the specification's definitions. For
example, `FieldSelectionMap` is a built-in scalar that represents a selection of
fields as a string. Redefining these built-in types with a different kind (e.g.,
an input object, enum, union, or object type) is disallowed and makes the
composition invalid.

To ensure schema evolution and interoperability, directives may include
additional arguments, provided that all required arguments defined by the
specification are present.

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

In the following example, the `@key` directive includes an additional argument,
`futureArg`, which is not part of the specification. This is valid and allows
the directive to evolve without breaking existing schemas.

```graphql example
directive @key(
  fields: FieldSelectionSet!
  futureArg: String
) repeatable on OBJECT | INTERFACE
```

However, if the `@key` directive is defined without the required `fields`
argument, as shown below, it results in a `TYPE_DEFINITION_INVALID` error.

```graphql counter-example
directive @key(futureArg: String) repeatable on OBJECT | INTERFACE
```

#### Query Root Type Inaccessible

**Error Code**

`QUERY_ROOT_TYPE_INACCESSIBLE`

**Severity**

ERROR

**Formal Specification**

- Let {queryType} be the query operation type defined in the schema.
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
schema {
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
schema {
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

#### Root Mutation Used

**Error Code**

`ROOT_MUTATION_USED`

**Severity**

ERROR

**Formal Specification**

- Let {rootMutation} be the root mutation type defined in the schema, if it
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

- Let {rootQuery} be the root query type defined in the schema, if it exists.
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

- Let {rootSubscription} be the root mutation type defined in the schema, if it
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

### Validate External Directives

#### External Unused

**Error Code**

`EXTERNAL_UNUSED`

**Severity**

ERROR

**Formal Specification**

- Let {schema} be the source schema to validate.
- Let {types} be the set of all composite types (object, interface) in {schema}.
- For each {type} in {types}:
  - Let {fields} be the set of fields for {type}.
  - For each {field} in {fields}:
    - If {field} is marked with `@external`:
      - Let {keyReferences} be the set of `@key` directives on types in {schema}
        whose `fields` selection selects {field}, including through nested
        selections.
      - Let {providesReferences} be the set of `@provides` directives on fields
        in {schema} whose `fields` selection selects {field}, including through
        nested selections.
      - The union of {keyReferences} and {providesReferences} must not be empty.

**Explanatory Text**

A field marked with `@external` is not resolved by the declaring source schema;
it is declared so that the source schema can reference it for composition
purposes. There are exactly two such purposes: entity identification, where the
field is selected by a `@key` directive, and field provision, where the field is
selected by a `@provides` directive. An `@external` field that is referenced by
neither `@key` nor `@provides` serves no purpose and is likely a leftover from
an incomplete refactoring; this rule reports it as an error.

Note: `@require` and `@is` express requirements through a `FieldSelectionMap`
that is resolved against data provided by other source schemas; they do not rely
on a local `@external` field declaration. References within `@require` or `@is`
therefore do not count as usage of an `@external` field.

**Examples**

In this example, the `name` field is marked with `@external` and is referenced
by the `@provides` directive, satisfying the rule:

```graphql example
# Source Schema A
type Product {
  id: ID
  name: String @external
}

type Query {
  productByName(name: String): Product @provides(fields: "name")
}
```

In this example, the `sku` and `upc` fields are marked with `@external` and are
each referenced by a `@key` directive on their declaring type, satisfying the
rule:

```graphql example
# Source schema A
type Product @key(fields: "sku") @key(fields: "upc") {
  sku: String! @external
  upc: String! @external
  name: String
}

type Query {
  productBySku(sku: String!): Product @lookup
  productByUpc(upc: String!): Product @lookup
}
```

In this example, the `name` field is marked with `@external` but is referenced
by neither a `@key` directive nor a `@provides` directive, violating the rule:

```graphql counter-example
# Source Schema A
type Product {
  id: ID
  name: String @external
}
```

#### External Override Collision

**Error Code**

`EXTERNAL_OVERRIDE_COLLISION`

**Severity**

ERROR

**Formal Specification**

- Let {schema} be the source schema to validate.
- Let {types} be the set of all {INTERFACE} and {OBJECT} types in {schema}.
- For each {type} in {types}:
  - Let {fields} be the set of fields on {type}.
  - For each {field} in {fields}:
    - If {field} is annotated with `@external`:
      - {field} must **not** be annotated with `@override`

**Explanatory Text**

The `@external` directive indicates that a field is **defined** in a different
source schema, and the current schema merely references it. Therefore, a field
marked with `@external` must **not** simultaneously carry directives that assume
local ownership or resolution responsibility, such as `@override`, which
transfers ownership of the field's definition from one schema to another, and is
incompatible with an already-external field definition.

**Examples**

In this scenario, `User.fullName` is defined in **Schema A** but overridden in
**Schema B**. Since `@override` is **not** combined with `@external` on the same
field, no collision occurs.

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
`EXTERNAL_OVERRIDE_COLLISION` error.

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

#### External Provides Collision

**Error Code**

`EXTERNAL_PROVIDES_COLLISION`

**Severity**

ERROR

**Formal Specification**

- Let {schema} be the source schema to validate.
- Let {types} be the set of all {INTERFACE} and {OBJECT} types in {schema}.
- For each {type} in {types}:
  - Let {fields} be the set of fields on {type}.
  - For each {field} in {fields}:
    - If {field} is annotated with `@external`:
      - {field} must **not** be annotated with `@provides`

**Explanatory Text**

The `@external` directive indicates that a field is **defined** in a different
source schema, and the current schema merely references it. Therefore, a field
marked with `@external` must **not** simultaneously carry directives that assume
local ownership or resolution responsibility, such as `@provides`, which
declares that the field can supply additional nested fields from the local
schema, conflicting with the notion of an external field whose definition
resides elsewhere.

**Examples**

In this example, `description` is **only** annotated with `@provides` in Schema
B, without any other directive. This usage is valid.

```graphql example
# Source Schema A
type Invoice {
  id: ID!
  description: String
}

# Source Schema B
type Invoice {
  id: ID!
  description: String @provides(fields: "length")
}
```

In this counter-example, `description` is annotated with `@external` and also
with `@provides`. Because `@external` and `@provides` cannot co-exist on the
same field, an `EXTERNAL_PROVIDES_COLLISION` error is produced.

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

#### External Require Collision

**Error Code**

`EXTERNAL_REQUIRE_COLLISION`

**Severity**

ERROR

**Formal Specification**

- Let {schema} be the source schema to validate.
- Let {types} be the set of all {INTERFACE} and {OBJECT} types in {schema}.
- For each {type} in {types}:
  - Let {fields} be the set of fields on {type}.
  - For each {field} in {fields}:
    - If {field} is annotated with `@external`:
      - For each {argument} in {field}:
        - {argument} must **not** be annotated with `@require`

**Explanatory Text**

The `@external` directive indicates that a field is **defined** in a different
source schema, and the current schema merely references it. Therefore, a field
marked with `@external` must **not** simultaneously carry directives that assume
local ownership or resolution responsibility, such as `@require`, which
specifies dependencies on other fields to resolve this field. Since `@external`
fields are not locally resolved, there is no need for `@require`.

**Examples**

In this example, `title` has arguments annotated with `@require` in Schema B,
but is not marked as `@external`. This usage is valid.

```graphql example
# Source Schema A
type Book {
  id: ID!
  title: String
  subtitle: String
}

# Source Schema B
type Book {
  id: ID!
  title(subtitle: String @require(field: "subtitle")): String
}
```

The following example is invalid, since `title` is marked with `@external` and
has an argument that is annotated with `@require`. This conflict leads to an
`EXTERNAL_REQUIRE_COLLISION` error.

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
  title(subtitle: String @require(field: "subtitle")): String @external
}
```

#### External on Interface

**Error Code**

`EXTERNAL_ON_INTERFACE`

**Severity**

ERROR

**Formal Specification**

- Let {schema} be the source schema to validate.
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

### Validate `@is` Directive

#### Is Invalid Syntax

**Error Code**

`IS_INVALID_SYNTAX`

**Severity**

ERROR

**Formal Specification**

- Let {types} be the set of all {INTERFACE} and {OBJECT} types in the source
  schema.
- For each {type} in {types}:
  - Let {fields} be the set of all lookup fields on {type}.
  - Let {arguments} be the set of all arguments on {fields}.
  - For each {argument} in {arguments}:
    - If {argument} is annotated with `@is`:
      - Let {fieldArg} be the string value of the `field` argument of the `@is`
        directive on {argument}.
      - {fieldArg} must be be parsable as a valid {FieldSelectionMap}.

**Explanatory Text**

The `@is` directive’s `field` argument must be syntactically valid GraphQL. If
the {FieldSelectionMap} string is malformed (e.g., missing closing braces,
unbalanced quotes, invalid tokens), then the schema cannot be composed
correctly. In such cases, the error `IS_INVALID_SYNTAX` is raised.

**Examples**

In the following example, the `@is` directive’s `field` argument is a valid
{FieldSelectionMap} and satisfies the rule.

```graphql example
type Query {
  product(id: ID! @is(field: "id")): Product @lookup
}

type Product {
  id: ID!
  name: String
}
```

In the following counter-example, the `@is` directive’s `field` argument has
invalid syntax because it is missing a closing brace.

```graphql counter-example
type Query {
  product(id: ID! @is(field: "{ id ")): Product @lookup
}

type Product {
  id: ID!
  name: String
}
```

#### Is Invalid Field Type

**Error Code**

`IS_INVALID_FIELD_TYPE`

**Severity**

ERROR

**Formal Specification**

- Let {schema} be the source schema to validate.
- Let {compositeTypes} be the set of all composite types in {schema}.
- For each {composite} in {compositeTypes}:
  - Let {fields} be the set of fields on {composite}.
  - Let {arguments} be the set of all arguments on {fields}.
  - For each {argument} in {arguments}:
    - If {argument} is **not** annotated with `@is`:
      - Continue
    - Let {fieldArg} be the value of the `field` argument of the `@is` directive
      on {argument}.
    - If {fieldArg} is **not** a string:
      - Produce an `IS_INVALID_FIELD_TYPE` error.

**Explanatory Text**

When using the `@is` directive, the `field` argument must always be a string
that describes how the arguments can be mapped from the _entity_ type that the
lookup field resolves. If the `field` argument is provided as a type other than
a string (such as an integer, boolean, or enum), the directive usage is invalid
and will cause schema composition to fail.

**Examples**

In the following example, the `@is` directive’s `field` argument is a valid
string and satisfies the rule.

```graphql example
type Query {
  personById(id: ID! @is(field: "id")): Person @lookup
}

type Person {
  id: ID!
  name: String
}
```

Since `field` is set to `123` (an integer) instead of a string, this violates
the rule and triggers an `IS_INVALID_FIELD_TYPE` error.

```graphql counter-example
type Query {
  personById(id: ID! @is(field: 123)): Person @lookup
}

type Person {
  id: ID!
  name: String
}
```

#### Is Invalid Usage

**Error Code**

`IS_INVALID_USAGE`

**Severity**

ERROR

**Formal Specification**

- Let {schema} be the source schema to validate.
- Let {compositeTypes} be the set of all composite types in {schema}.
- For each {compositeType} in {compositeTypes}:
  - Let {fields} be the set of fields on {compositeType}.
  - For each {field} in {fields}:
    - Let {arguments} be the set of all arguments on {field}.
    - For each {argument} in {arguments}:
      - If {argument} is **not** annotated with `@is`:
        - Continue
      - {field} must be annotated with `@lookup`

**Explanatory Text**

When using the `@is` directive, the field declaring the argument must be a
lookup field (i.e. have the `@lookup` directive applied).

**Examples**

In the following example, the `@is` directive is applied to an argument declared
on a field with the `@lookup` directive, satisfying the rule.

```graphql example
type Query {
  personById(id: ID! @is(field: "id")): Person @lookup
}

type Person {
  id: ID!
  name: String
}
```

In the following counter-example, the `@is` directive is applied to an argument
declared on a field without the `@lookup` directive, violating the rule.

```graphql counter-example
type Query {
  personById(id: ID! @is(field: "id")): Person
}

type Person {
  id: ID!
  name: String
}
```

#### Is Fields Has Arguments

**Error Code**

`IS_FIELDS_HAS_ARGUMENTS`

**Severity**

ERROR

**Formal Specification**

- Let {schema} be the source schema to validate.
- Let {compositeTypes} be the set of all composite types in {schema}.
- For each {compositeType} in {compositeTypes}:
  - Let {fields} be the set of fields on {compositeType}.
  - For each {field} in {fields}:
    - Let {arguments} be the set of arguments on {field}.
    - For each {argument} in {arguments}:
      - If {argument} is **not** annotated with `@is`:
        - Continue
      - Let {selectionMap} be the parsed selection map of the `field` argument
        of the `@is` directive on {argument}.
      - Each selection in {selectionMap}, including nested selections and path
        segments, must **not** supply arguments.

**Explanatory Text**

The arguments of a lookup field represent the _stable key_ with which the
_distributed GraphQL executor_ recalls an _entity_, and the `@is` directive maps
each argument to a field of the _entity_. Such a mapping must consist of plain
field paths; supplying arguments within an `@is` selection map is not allowed.

A referenced field may still declare arguments, as long as each argument is
nullable, has a default value, or is annotated with `@require` and therefore
supplied by the executor. Such a field can be resolved without any arguments
being supplied. A field that requires an argument cannot be referenced by an
`@is` selection map, since the map cannot supply one (see
[Is Invalid Fields](#sec-Is-Invalid-Fields) and the argument validation rules in
Appendix A).

The same applies to `@key` (see
[Key Fields Has Arguments](#sec-Key-Fields-Has-Arguments)). `@provides`
selections must reference fields that declare no arguments other than
`@require`-annotated ones (see
[Provides Fields Has Arguments](#sec-Provides-Fields-Has-Arguments)). Only
`@require` selection maps may supply constant arguments, as they derive input
values rather than keys.

**Examples**

In this example, the `id` argument of the lookup field is mapped to the plain
`id` field of `Product`, satisfying the rule.

```graphql example
type Query {
  productById(id: ID! @is(field: "id")): Product @lookup
}

type Product {
  id: ID!
  name: String
}
```

In this example, the referenced field `id` declares the nullable `scope`
argument. Since `scope` can be omitted, `id` can be resolved without supplying
an argument and may be referenced by the `@is` selection map.

```graphql example
type Query {
  productById(id: ID! @is(field: "id")): Product @lookup
}

type Product {
  id(scope: IdScope): ID!
  name: String
}
```

In this counter-example, the `@is` selection map supplies an argument on `id`,
violating the rule.

```graphql counter-example
type Query {
  productByLocalId(id: ID! @is(field: "id(scope: LOCAL)")): Product @lookup
}

type Product {
  id(scope: IdScope): ID!
  name: String
}
```

### Validate Key Directives

#### Key Fields Select Invalid Type

**Error Code**

`KEY_FIELDS_SELECT_INVALID_TYPE`

**Severity**

ERROR

**Formal Specification**

- Let {types} be the set of all object or interface types that are annotated
  with the `@key` directive in the schema.
- For each {type} in {types}:
  - Let {keyDirectives} be the set of all `@key` directives on {type}.
  - For each {keyDirective} in {keyDirectives}
    - Let {keyFields} be the set of all fields (including nested) referenced by
      the `fields` argument of {keyDirective}.
    - For each {field} in {keyFields}:
      - Let {fieldType} be the type of {field}.
      - {fieldType} must not be a `List`, `Interface`, or `Union` type.

**Explanatory Text**

The `@key` directive is used to define the set of fields that uniquely identify
an _entity_. These fields must reference scalars or object types to ensure a
valid and consistent representation of the _entity_ across schemas. Fields of
types `List`, `Interface`, or `Union` cannot be part of a `@key` because they do
not have a well-defined unique value.

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

`KEY_DIRECTIVE_IN_FIELDS_ARGUMENT`

**Severity**

ERROR

**Formal Specification**

- Let {types} be the set of all object and interface types in the schema.
- For each {type} in {types}:
  - Let {keyDirectives} be the set of all `@key` directives on {type}.
  - For each {keyDirective} in {keyDirectives}:
    - Let {fields} be the string value of the `fields` argument of
      {keyDirective}.
    - {fields} must not contain a directive application.

**Explanatory Text**

The `@key` directive specifies the set of fields used to uniquely identify an
_entity_. The `fields` argument must consist of a valid GraphQL selection set
that does not include any directive applications. Directives in the `fields`
argument are not supported.

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

`KEY_FIELDS_HAS_ARGUMENTS`

**Severity**

ERROR

**Formal Specification**

- Let {types} be the set of all object and interface types in the schema that
  are annotated with the `@key` directive.
- For each {type} in {types}:
  - Let {keyDirectives} be the set of all `@key` directives on {type}.
  - For each {keyDirective} in {keyDirectives}:
    - Let {selections} be the field selections of the `fields` argument of
      {keyDirective}.
    - For each {selection} in {selections}:
      - {KeyFieldsHasArguments(selection, type)} must be false.

KeyFieldsHasArguments(selection, type):

- Let {field} be the field of {type} selected by {selection}.
- If {selection} supplies arguments:
  - return true
- For each {argumentDefinition} declared by {field}:
  - If the type of {argumentDefinition} is Non-Null, {argumentDefinition} has no
    default value, and {argumentDefinition} is not annotated with `@require`:
    - return true
- If {selection} has a selection set:
  - Let {subType} be the return type of {field}.
  - Let {subSelections} be the selections in the selection set of {selection}.
  - For each {subSelection} in {subSelections}:
    - If {KeyFieldsHasArguments(subSelection, subType)} is true:
      - return true
- return false

**Explanatory Text**

The `@key` directive designates the fields that form a _stable key_ of an
_entity_. Selections within the `fields` argument must not supply arguments: a
_stable key_ must consist of plain field values that identify an _entity_
deterministically, and the _distributed GraphQL executor_ resolves key fields
without supplying any arguments.

A referenced field may still declare arguments, as long as each argument is
nullable, has a default value, or is annotated with `@require` and therefore
supplied by the executor. Such a field can be resolved without any arguments
being supplied. A field that requires an argument cannot be part of a _stable
key_ because it cannot be resolved without one.

The same applies to `@is` (see
[Is Fields Has Arguments](#sec-Is-Fields-Has-Arguments)). `@provides` selections
must reference fields that declare no arguments other than `@require`-annotated
ones (see [Provides Fields Has Arguments](#sec-Provides-Fields-Has-Arguments)).
Only `@require` selection maps may supply constant arguments, as they derive
input values rather than keys.

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

In this example, the `@key` directive references the field `tags`, which
declares the optional `limit` argument. Since `limit` can be omitted, `tags` can
be resolved without supplying an argument and may be part of the key.

```graphql example
type User @key(fields: "id tags") {
  id: ID!
  tags(limit: Int = 10): [String]
}
```

In this counter-example, the key selection supplies arguments on `id`.
Selections within the `fields` argument must not supply arguments.

```graphql counter-example
type User @key(fields: "id(scope: LOCAL)") {
  id: ID!
}
```

In this counter-example, the `@key` directive references the field `tags`, which
requires the `limit` argument. Since `limit` can neither be omitted nor supplied
by the key selection, `tags` cannot be part of a key.

```graphql counter-example
type User @key(fields: "id tags") {
  id: ID!
  tags(limit: Int!): [String]
}
```

#### Key Invalid Syntax

**Error Code**

`KEY_INVALID_SYNTAX`

**Severity**

ERROR

**Formal Specification**

- Let {types} be the set of all object or interface types in the schema.
- For each {type} in {types}:
  - Let {keyDirectives} be the set of all `@key` directives on {type}.
  - For each {keyDirective} in {keyDirectives}:
    - Let {fieldsArg} be the string value of the `fields` argument of
      {keyDirective}.
    - Attempt to parse {fieldsArg} as a valid GraphQL selection set.
    - Parsing must **not** fail (e.g., missing braces, invalid tokens,
      unbalanced curly braces, or other syntax errors).

**Explanatory Text**

Each `@key` directive must specify the fields that uniquely identify an _entity_
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

- Let {types} be the set of all object and interface types in the schema.
- For each {type} in {types}:
  - Let {keyDirectives} be the set of all `@key` directives on {type}.
  - For each {keyDirective} in {keyDirectives}:
    - Let {fieldsArg} be the string value of the `fields` argument of
      {keyDirective}.
    - Let {selections} be the set of fields in the selection set of {fieldsArg}.
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
named, composition fails with a `KEY_INVALID_FIELDS` error because the _stable
key_ cannot be resolved correctly.

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

#### Key Invalid Fields Type

**Error Code**

`KEY_INVALID_FIELDS_TYPE`

**Severity**

ERROR

**Formal Specification**

- Let {schema} be the source schema to validate.
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
`"id uuid"`, identifying two fields that form a composite _stable key_. This
usage is valid.

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

#### Interface Object Key Missing

**Error Code**

`INTERFACE_OBJECT_KEY_MISSING`

**Severity**

ERROR

**Formal Specification**

- Let {schema} be the source schema to validate.
- Let {types} be the set of all object types in {schema} annotated with
  `@interfaceObject`.
- For each {type} in {types}:
  - Let {keyDirectives} be the set of all `@key` directives on {type}.
  - {keyDirectives} must not be empty.

**Explanatory Text**

An object type annotated with `@interfaceObject` stands in for an interface
defined in one or more other source schemas. The composite schema resolves this
stand-in as an independently queryable entity. The _distributed executor_ must
be able to fetch it by its key, or extend it with the fields it contributes. The
type must therefore declare at least one `@key`, exactly as any other entity
type would. A stand-in with no key cannot be targeted by the executor and cannot
contribute fields to the interface it stands in for.

This rule only requires that a key exists. The fields selected by the key are
validated like those of any other `@key` (see
[Validate Key Directives](#sec-Validate-Key-Directives)). Whether the key
matches one of the keys declared on the interface is validated across source
schemas by [Interface Object Key Mismatch](#sec-Interface-Object-Key-Mismatch).

**Examples**

In this example, the `Media` stand-in declares a `@key`, so source schema B's
contribution can be joined to the `Media` interface by `id`.

```graphql example
# Source Schema B
type Media @interfaceObject @key(fields: "id") {
  id: ID!
  reviews: [Review!]!
}

type Review {
  id: ID!
  rating: Int!
}
```

In the following counter-example, the `Media` stand-in declares no `@key`, so
the entity it stands in for cannot be resolved. This results in an
`INTERFACE_OBJECT_KEY_MISSING` error.

```graphql counter-example
# Source Schema B
type Media @interfaceObject {
  id: ID!
  reviews: [Review!]!
}

type Review {
  id: ID!
  rating: Int!
}
```

### Validate Lookup Directives

#### Lookup Must Have Arguments

**Error Code**

`LOOKUP_MUST_HAVE_ARGUMENTS`

**Severity**

ERROR

**Formal Specification**

- Let {schema} be the source schema to validate.
- Let {fields} be the set of all field definitions annotated with `@lookup` in
  {schema}.
- For each {field} in {fields}:
  - The number of arguments on {field} must be greater than zero.

**Explanatory Text**

Fields annotated with the `@lookup` directive identify a single _entity_ by the
arguments supplied to them. A lookup field that declares no arguments has no
_stable key_ with which to resolve an _entity_ and cannot participate in
composition. This rule reports such fields as invalid.

**Examples**

For example, the following usage is valid because `productById` declares an
argument that can be used to resolve a `Product` _entity_.

```graphql example
type Query {
  productById(id: ID!): Product @lookup
}

type Product {
  id: ID!
  name: String
}
```

This counter-example demonstrates an invalid usage. The `product` field is
annotated with `@lookup` but declares no arguments, so it cannot identify which
_entity_ to resolve.

```graphql counter-example
type Query {
  product: Product @lookup
}

type Product {
  id: ID!
  name: String
}
```

#### Lookup Returns Non-Nullable Type

**Error Code**

`LOOKUP_RETURNS_NON_NULLABLE_TYPE`

**Severity**

WARNING

**Formal Specification**

- Let {schema} be the source schema to validate.
- Let {fields} be the set of all field definitions annotated with `@lookup` in
  {schema}.
- For each {field} in {fields}:
  - Let {type} be the return type of {field}.
  - {type} must be a nullable type.

**Explanatory Text**

Fields annotated with the `@lookup` directive are intended to retrieve a single
_entity_ based on provided arguments. To properly handle cases where the
requested _entity_ does not exist, such fields should have a nullable return
type. This allows the field to return `null` when an _entity_ matching the
provided criteria is not found, following the standard GraphQL practices for
representing missing data.

In a distributed system, it is likely that some _entities_ will not be found on
other schemas, even when those schemas contribute fields to the type. Ensuring
that `@lookup` fields have nullable return types also avoids GraphQL errors on
schemas and prevents result erasure through non-null propagation. By allowing
null to be returned when an _entity_ is not found, the system can gracefully
handle missing data without causing exceptions or unexpected behavior.

Ensuring that `@lookup` fields have nullable return types allows gateways to
distinguish between cases where an _entity_ is not found (receiving null) and
other error conditions that may have to be propagated to the client.

For example, the following usage is recommended:

```graphql example
type Query {
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
type Query {
  userById(id: ID!): User! @lookup
}

type User {
  id: ID!
  name: String
}
```

Here, `userById` returns a non-nullable `User!`, which does not align with the
recommendation that a `@lookup` field should have a nullable return type.

#### Lookup Returns List

**Error Code**

`LOOKUP_RETURNS_LIST`

**Severity**

ERROR

**Formal Specification**

- Let {schema} be the source schema to validate.
- Let {fields} be the set of all field definitions annotated with `@lookup` in
  {schema}.
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
_entity_ based on provided arguments. To avoid ambiguity in _entity_ resolution,
such fields must return a single object and not a list. This validation rule
enforces that any field annotated with `@lookup` must have a return type that is
**NOT** a list.

**Examples**

For example, the following usage is valid:

```graphql example
type Query {
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
type Query {
  usersByIds(ids: [ID!]!): [User!] @lookup
}

type User {
  id: ID!
  name: String
}
```

Here, `usersByIds` returns a list of `User` objects, which violates the
requirement that a `@lookup` field must return a single object.

#### Lookup Key Missing For Type

**Error Code**

`LOOKUP_KEY_MISSING_FOR_TYPE`

**Severity**

ERROR

**Formal Specification**

- Let {schema} be the source schema to validate.
- Let {lookupFields} be the set of all fields in {schema} annotated with
  `@lookup`.
- For each {lookupField} in {lookupFields}:
  - Let {returnType} be the unwrapped return type of {lookupField}.
  - Let {possibleTypes} be {GetPossibleTypes(returnType)}.
  - For each {argument} in the arguments of {lookupField}:
    - For each {possibleType} in {possibleTypes}:
      - {IsArgumentMappable(argument, possibleType)} must be true.

IsArgumentMappable(argument, possibleType):

- If {argument} is annotated with `@is`:
  - Let {selectionMap} be the parsed selection map of the `field` argument of
    the `@is` directive on {argument}.
  - For each {alternative} in the alternatives of {selectionMap}:
    - If the type conditions of {alternative} admit {possibleType} and all
      fields referenced by {alternative} are defined for {possibleType}:
      - return true
  - return false
- Otherwise:
  - Let {argumentName} be the name of {argument}.
  - If {possibleType} defines a field named {argumentName}:
    - return true
  - return false

**Explanatory Text**

A lookup field must be able to resolve every possible runtime type of its return
type. The arguments of a lookup field represent the stable key with which an
entity is resolved, and each argument must independently be mappable to a field
for every possible object type of the return type.

Without an `@is` directive, an argument is mapped by its name: every possible
object type must define a field with the argument's name, either directly or
through an interface. With an `@is` directive, the selection map defines the
mapping, and its alternatives may map different runtime types to different
fields. The alternatives must still cover every possible type: a runtime type
that is not covered by any alternative cannot be resolved by the lookup field. A
source schema that can only resolve a subset of the possible types must declare
a narrower return type instead.

**Examples**

In this example, the selection map covers all possible types of `Media`,
resolving each by a different key field.

```graphql example
type Query {
  mediaByKey(
    key: MediaKeyInput!
      @is(
        field: "{ isbn: <Book>.isbn } | { upc: <Movie>.upc } | { feedUrl: <Podcast>.feedUrl }"
      )
  ): Media @lookup
}

input MediaKeyInput @oneOf {
  isbn: String
  upc: String
  feedUrl: String
}

interface Media {
  id: ID!
}

type Book implements Media {
  id: ID!
  isbn: String!
}

type Movie implements Media {
  id: ID!
  upc: String!
}

type Podcast implements Media {
  id: ID!
  feedUrl: String!
}
```

In this counter-example, the selection map covers only `Book` and `Movie`.
`Podcast` is a possible type of `Media` but is not covered by any alternative,
violating the rule.

```graphql counter-example
type Query {
  mediaByKey(
    key: MediaKeyInput!
      @is(field: "{ isbn: <Book>.isbn } | { upc: <Movie>.upc }")
  ): Media @lookup
}

input MediaKeyInput @oneOf {
  isbn: String
  upc: String
}

interface Media {
  id: ID!
}

type Book implements Media {
  id: ID!
  isbn: String!
}

type Movie implements Media {
  id: ID!
  upc: String!
}

type Podcast implements Media {
  id: ID!
  feedUrl: String!
}
```

In this counter-example, the arguments are mapped by name, but the possible type
`Clothing` does not define a field named `categoryId`, violating the rule.

```graphql counter-example
type Query {
  product(id: ID!, categoryId: Int): Product @lookup
}

union Product = Electronics | Clothing

type Electronics {
  id: ID!
  categoryId: Int
  name: String
}

type Clothing {
  id: ID!
  name: String
}
```

### Validate Override Directives

#### Override from Self

**Error Code**

`OVERRIDE_FROM_SELF`

**Severity**

ERROR

**Formal Specification**

- Let {schema} be the source schema to validate.
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

- Let {schema} be the source schema to validate.
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

### Validate Provides Directives

#### Provides Directive in Fields Argument

**Error Code**

`PROVIDES_DIRECTIVE_IN_FIELDS_ARGUMENT`

**Severity**

ERROR

**Formal Specification**

- Let {fieldsWithProvides} be the set of all fields annotated with the
  `@provides` directive in the schema.
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

`PROVIDES_FIELDS_HAS_ARGUMENTS`

**Severity**

ERROR

**Formal Specification**

- Let {fieldsWithProvides} be the set of all fields annotated with the
  `@provides` directive in the schema.
- For each {field} in {fieldsWithProvides}:
  - Let {selections} be the field selections of the `fields` argument of the
    `@provides` directive on {field}.
  - Let {type} be the return type of {field}.
  - For each {selection} in {selections}:
    - {ProvidesHasArguments(selection, type)} must be false.

ProvidesHasArguments(selection, type):

- Let {field} be the field of {type} selected by {selection}.
- If {field} declares an argument that is not annotated with `@require`:
  - return true
- If {selection} supplies arguments:
  - return true
- If {selection} has a selection set:
  - Let {subType} be the return type of {field}.
  - Let {subSelections} be the selections in the selection set of {selection}.
  - For each {subSelection} in {subSelections}:
    - If {ProvidesHasArguments(subSelection, subType)} is true:
      - return true
- return false

**Explanatory Text**

The `@provides` directive specifies fields that a resolver provides for the
parent type. The `fields` argument must reference fields that do not declare
arguments, as fields with arguments introduce variability that is incompatible
with the consistent behavior expected of `@provides`. Arguments annotated with
`@require` are exempt, since they are supplied by the executor rather than
chosen by the consumer. A selection within the `fields` argument must never
supply arguments.

Note: Unlike `@require`, which describes how to derive a value (and may
therefore include constant arguments to disambiguate the selection), `@provides`
advertises that the resolver returns the listed fields as part of its parent's
selection set. Because the consumer chooses the arguments at query time, the
resolver cannot pre-commit to a specific parameterization, and constant
arguments would be meaningless here.

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

In this example, the `tags` field declares only an argument annotated with
`@require`. Since its value is supplied by the executor, `tags` may still be
referenced by the `@provides` selection.

```graphql example
type User @key(fields: "id") {
  id: ID!
  tags(limit: Int @require(field: "tagLimit")): [String]
  tagLimit: Int
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

In this counter-example, the `@provides` selection supplies arguments on `tags`,
even though `tags` does not declare any. Selections within the `fields` argument
must not supply arguments.

```graphql counter-example
type User @key(fields: "id") {
  id: ID!
  tags: [String]
}

type Article @key(fields: "id") {
  id: ID!
  author: User! @provides(fields: "tags(limit: 10)")
}
```

#### Provides Fields Missing External

**Error Code**

`PROVIDES_FIELDS_MISSING_EXTERNAL`

**Severity**

ERROR

**Formal Specification**

- Let {objectTypes} be the set of all object types in the schema.
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

#### Provides Invalid Syntax

**Error Code**

`PROVIDES_INVALID_SYNTAX`

**Severity**

ERROR

**Formal Specification**

- Let {fieldsWithProvides} be the set of all fields annotated with the
  `@provides` directive in the schema.
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

#### Provides Invalid Fields

**Error Code**

`PROVIDES_INVALID_FIELDS`

**Severity**

ERROR

**Formal Specification**

- Let {schema} be the source schema to validate.
- Let {fieldsWithProvides} be the set of all fields annotated with the
  `@provides` directive in {schema}.
- For each {field} in {fieldsWithProvides}:
  - Let {fieldsArg} be the string value of the `fields` argument of the
    `@provides` directive on {field}.
  - Let {parsedFieldSelectionSet} be the parsed field selection set from
    {fieldsArg}.
  - Let {returnType} be the return type of {field}.
  - {ValidateFieldSelectionSet(parsedFieldSelectionSet, returnType)} must be
    true.

ValidateFieldSelectionSet(fieldSelectionSet, parentType):

- For each {selection} in {fieldSelectionSet}:
  - Let {selectedField} be the field selected by {selection} in {parentType}.
  - If {selectedField} does not exist on {parentType}:
    - return false
  - Let {selectedType} be the type of {selectedField}
  - If {selectedType} is an {INTERFACE} or {OBJECT} type
    - Let {subSelectionSet} be the field selection set of {selection}
    - If {subSelectionSet} is empty
      - return false
    - If {ValidateFieldSelectionSet(subSelectionSet, fieldType)} is false
      - return false
- return true

**Explanatory Text**

Even if the `@provides(fields: "…")` argument is well-formed syntactically, the
selected fields must actually exist on the return type of the field. Invalid
field references—e.g., selecting non-existent fields, referencing fields on the
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

#### Provides Invalid Fields Type

**Error Code**

`PROVIDES_INVALID_FIELDS_TYPE`

**Severity**

ERROR

**Formal Specification**

- Let {schema} be the source schema to validate.
- Let {types} be the set of all composite types in {schema}.
- For each {type} in {types}:
  - Let {fields} be the set of fields on {type}.
  - For each {field} in {fields}:
    - If {field} is annotated with `@provides`:
      - Let {fieldsArg} be the value of the `fields` argument on the `@provides`
        directive.
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

- Let {schema} be the source schema to validate.
- Let {types} be the set of all object and interface types in {schema}.
- For each {type} in {types}:
  - Let {fields} be the set of fields on {type}.
  - For each {field} in {fields}:
    - If {field} is annotated with `@provides`:
      - Let {fieldType} be the base return type of {field} (i.e., unwrapped of
        any `[ ]` or `!`).
      - {fieldType} must be an interface or object type.

**Explanatory Text**

The `@provides` directive allows a field to “provide” additional nested fields
on the composite type it returns. If a field's base type is not an object or
interface type (e.g., `String`, `Int`, `Boolean`, `Enum`, `Union`, or an `Input`
type), it cannot hold nested fields for `@provides` to select. Consequently,
attaching `@provides` to such a field is invalid and raises a
`PROVIDES_ON_NON_COMPOSITE_FIELD` error.

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
`PROVIDES_ON_NON_COMPOSITE_FIELD` error.

```graphql counter-example
type User {
  id: ID!
  email: String @provides(fields: "length")
}
```

### Validate Require Directives

#### Require Invalid Syntax

**Error Code**

`REQUIRE_INVALID_SYNTAX`

**Severity**

ERROR

**Formal Specification**

- Let {compositeTypes} be the set of all composite types in the schema.
- For each {composite} in {compositeTypes}:
  - Let {fields} be the set of fields on {composite}.
  - Let {arguments} be the set of all arguments on {fields}.
  - For each {argument} in {arguments}:
    - If {argument} is **not** annotated with `@require`:
      - Continue
    - Let {fieldArg} be the string value of the `field` argument of the
      `@require` directive on {argument}.
    - {fieldArg} must be be parsable as a valid selection map

**Explanatory Text**

The `@require` directive's `field` argument must be syntactically valid GraphQL.
If the selection map string is malformed (e.g., missing closing braces,
unbalanced quotes, invalid tokens), then the schema cannot be composed
correctly. In such cases, the error `REQUIRE_INVALID_SYNTAX` is raised.

**Examples**

In the following example, the `@require` directive's `field` argument is a valid
selection map and satisfies the rule.

```graphql example
type User @key(fields: "id") {
  id: ID!
  profile(name: String @require(field: "name")): Profile
}

type Profile {
  id: ID!
  name: String
}
```

In the following counter-example, the `@require` directive's `field` argument
has invalid syntax because it is missing a closing brace.

This violates the rule and triggers a `REQUIRE_INVALID_SYNTAX` error.

```graphql counter-example
type User @key(fields: "id") {
  id: ID!
  profile(name: String! @require(field: "{ name ")): Profile
}

type Profile {
  id: ID!
  name: String
}
```

#### Require Invalid Fields Type

**Error Code**

`REQUIRE_INVALID_FIELD_TYPE`

**Severity**

ERROR

**Formal Specification**

- Let {schema} be the source schema to validate.
- Let {compositeTypes} be the set of all composite types in {schema}.
- For each {composite} in {compositeTypes}:
  - Let {fields} be the set of fields on {composite}.
  - Let {arguments} be the set of all arguments on {fields}.
  - For each {argument} in {arguments}:
    - If {argument} is **not** annotated with `@require`:
      - Continue
    - Let {fieldArg} be the value of the `field` argument of the `@require`
      directive on {argument}.
    - If {fieldArg} is **not** a string:
      - Produce a `REQUIRE_INVALID_FIELD_TYPE` error.

**Explanatory Text**

When using the `@require` directive, the `field` argument must always be a
string that defines a (potentially nested) selection set of fields from the same
type. If the `field` argument is provided as a type other than a string (such as
an integer, boolean, or enum), the directive usage is invalid and will cause
schema composition to fail.

**Examples**

In the following example, the `@require` directive's `field` argument is a valid
string and satisfies the rule.

```graphql example
type User @key(fields: "id") {
  id: ID!
  profile(name: String @require(field: "name")): Profile
}

type Profile {
  id: ID!
  name: String
}
```

Since `field` is set to `123` (an integer) instead of a string, this violates
the rule and triggers a `REQUIRE_INVALID_FIELD_TYPE` error.

```graphql counter-example
type User @key(fields: "id") {
  id: ID!
  profile(name: String! @require(field: 123)): Profile
}

type Profile {
  id: ID!
  name: String
}
```

#### Require Invalid Usage

**Error Code**

`REQUIRE_INVALID_USAGE`

**Severity**

ERROR

**Formal Specification**

- Let {schema} be the source schema to validate.
- Let {compositeTypes} be the set of all composite types in {schema}.
- For each {compositeType} in {compositeTypes}:
  - Let {fields} be the set of fields on {compositeType}.
  - For each {field} in {fields}:
    - If {field} is annotated with `@lookup`:
      - Let {arguments} be the set of all arguments on {field}.
      - For each {argument} in {arguments}:
        - {argument} must **not** be annotated with `@require`

**Explanatory Text**

The arguments of a lookup field represent the stable key with which the
_distributed GraphQL executor_ resolves an entity. Their values are supplied
from an existing representation of the entity - either directly by argument name
or through an `@is` mapping - before the lookup is executed.

The `@require` directive, in contrast, expresses a data dependency of a field
that is resolved in the context of an existing parent object. A lookup field is
used to establish that context in the first place; for a lookup field reachable
from the root `Query` type, no parent entity exists from which a requirement
could be fulfilled. The satisfiability validation likewise describes lookup
inputs solely through `@is` mappings or argument names; an argument annotated
with `@require` has no defined contribution to a lookup.

Therefore, annotating an argument of a lookup field with `@require` is invalid
and raises a `REQUIRE_INVALID_USAGE` error.

**Examples**

In the following example, the lookup field `productById` resolves `Product` by
its stable key, and the requirement is declared on the argument of an ordinary
field, satisfying the rule.

```graphql example
# Source Schema A
type Query {
  productById(id: ID!): Product @lookup
}

type Product @key(fields: "id") {
  id: ID!
  shippingCost(weight: Float @require(field: "shippingWeight")): Currency
}

# Source Schema B
type Product @key(fields: "id") {
  id: ID!
  shippingWeight: Float
}
```

In the following counter-example, the `locale` argument of the lookup field
`productById` is annotated with `@require`, violating the rule.

```graphql counter-example
# Source Schema A
type Query {
  productById(
    id: ID!
    locale: String @require(field: "defaultLocale")
  ): Product @lookup
}

type Product @key(fields: "id") {
  id: ID!
}

# Source Schema B
type Query {
  defaultLocale: String
}
```

#### Require Inconsistent on Implementation

**Error Code**

`REQUIRE_INCONSISTENT_ON_IMPLEMENTATION`

**Severity**

ERROR

**Formal Specification**

- Let {schema} be the source schema to validate.
- Let {implementingTypes} be the set of all object and interface types in
  {schema} that implement at least one interface.
- For each {implementingType} in {implementingTypes}:
  - Let {interfaces} be the set of interface types that {implementingType}
    implements.
  - For each {interface} in {interfaces}:
    - Let {interfaceFields} be the set of fields on {interface}.
    - For each {interfaceField} in {interfaceFields}:
      - Let {implementingField} be the field on {implementingType} with the same
        name as {interfaceField}.
      - For each {interfaceArgument} in the arguments of {interfaceField}:
        - Let {implementingArgument} be the argument on {implementingField} with
          the same name as {interfaceArgument}.
        - If {interfaceArgument} is annotated with `@require`:
          - {implementingArgument} must be annotated with `@require`
        - Otherwise:
          - {implementingArgument} must **not** be annotated with `@require`

**Explanatory Text**

The `@require` directive may be applied to arguments of fields declared on
interface types. The selection map is rooted at the interface type and is
evaluated against the concrete runtime object: fields declared on the interface
can be selected without type conditions, while fields of specific implementing
types can be referenced through type conditions.

GraphQL requires an implementing field to redeclare every argument of the
interface field, and composition removes all arguments annotated with `@require`
from the composite schema. The `@require` annotation must therefore be applied
consistently across the interface contract: an argument is annotated with
`@require` on the interface field and on the corresponding argument of every
implementing field, or on neither. Consistent annotation removes the argument
from the interface field and from all implementing fields together, so the
composite schema retains a valid interface contract. Inconsistent annotation
would remove the argument from only one side of the contract and break the
composite schema.

The selection maps of the interface field argument and of an implementing field
argument may differ: each is validated against its own declaring type, and an
implementing type may derive the required value from implementation-specific
fields.

Note: Cross-schema cases in which a merged interface field declares an argument
that an implementing field lacks are detected after merging by
[Interface Field Argument No Implementation](#sec-Interface-Field-Argument-No-Implementation).

**Examples**

In this example, the `locale` argument is annotated with `@require` on the
interface field `Account.displayName` and on the implementing field
`User.displayName`, satisfying the rule.

```graphql example
# Source Schema A
interface Account {
  id: ID!
  displayName(locale: String @require(field: "preferredLocale")): String
}

type User implements Account @key(fields: "id") {
  id: ID!
  displayName(locale: String @require(field: "preferredLocale")): String
}

# Source Schema B
interface Account {
  id: ID!
  preferredLocale: String
}

type User implements Account @key(fields: "id") {
  id: ID!
  preferredLocale: String
}
```

In this counter-example, the `locale` argument is annotated with `@require` on
the implementing field `User.displayName` but not on the interface field
`Account.displayName`, violating the rule. The composite schema would declare
`locale` on the interface field but not on the implementing field, breaking the
interface contract.

```graphql counter-example
# Source Schema A
interface Account {
  id: ID!
  displayName(locale: String): String
}

type User implements Account @key(fields: "id") {
  id: ID!
  displayName(locale: String @require(field: "preferredLocale")): String
}

# Source Schema B
type User @key(fields: "id") {
  id: ID!
  preferredLocale: String
}
```

### Validate Shareable Directives

#### Invalid Shareable Usage

**Error Code**

`INVALID_SHAREABLE_USAGE`

**Severity**

ERROR

**Formal Specification**

- Let {schema} be the source schema to validate.
- Let {types} be the set of types defined in {schema}.
- For each {type} in {types}:
  - If {type} is an interface type:
    - For each field definition {field} in {type}:
      - If {field} is annotated with `@shareable`, produce an
        `INVALID_SHAREABLE_USAGE` error.
  - If {type} is the `Subscription` type:
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

Additionally, subscription root fields cannot be shared (i.e., they are
effectively non-shareable), as subscription events from multiple schemas would
create conflicts in the composed schema. Attempting to mark a subscription field
as shareable or to define it in multiple schemas triggers the same error.

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

In this counter-example, the `InventoryItem` interface has a field `sku` marked
with `@shareable`, which is invalid usage. Marking an interface field as
shareable leads to an `INVALID_SHAREABLE_USAGE` error.

```graphql counter-example
interface InventoryItem {
  sku: ID! @shareable
  name: String
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

## Pre Merge Validation

Prior to merging the schemas, additional validations are performed that require
visibility into all source schemas but treat each source schema separately. This
step detects conflicts such as incompatible fields or default argument values
that would render the merged schema unusable. Detecting such conflicts early
prevents errors that would otherwise be discovered during the merge process.

### Validate Type System

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
  - Let {consideredTypes} be the subset of {types} excluding any object type
    annotated with `@interfaceObject`.
  - All {consideredTypes} must be of the same kind (Object, Interface, Union,
    Enum, InputObject, Scalar).

**Explanatory Text**

Each named type must represent the **same** kind of GraphQL type across all
source schemas. For instance, a type named `User` must consistently be an object
type, or consistently be an interface, and so forth. If one schema defines
`User` as an object type, while another schema declares `User` as an interface
(or input object, union, etc.), the schema composition process cannot merge
these definitions coherently.

A single type name cannot represent two different kinds of type in the composed
schema.

The one exception is an object type annotated with `@interfaceObject`. Such a
type is a stand-in for an interface of the same name (see
[Interface Object No Interface](#sec-Interface-Object-No-Interface)).
Composition deliberately excludes it from this check. An `@interfaceObject` type
and the interface it stands in for are not a kind mismatch. They are the
mechanism by which a source schema contributes field implementations that
composition projects onto an interface's implementing types. An object type with
the same name that is **not** annotated with `@interfaceObject` is still an
ordinary kind mismatch.

**Examples**

All schemas agree that `User` is an object type:

```graphql example
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
in a `TYPE_KIND_MISMATCH` error.

```graphql counter-example
# Schema A: `User` is an object type
type User {
  id: ID!
  name: String
}

# Schema B: `User` is an interface
interface User {
  id: ID!
  friends: [User!]!
}
```

`Media` is declared as an interface in source schema A and as an object type
annotated with `@interfaceObject` in source schema B. Because the stand-in is
annotated, it is excluded from the kind check and no error is raised.

```graphql example
# Source Schema A: `Media` is an interface
interface Media {
  id: ID!
  title: String!
}

# Source Schema B: `Media` is an `@interfaceObject` stand-in
type Media @interfaceObject @key(fields: "id") {
  id: ID!
  reviews: [Review!]!
}

type Review {
  id: ID!
  rating: Int!
}
```

Here, source schema B declares `Media` as a plain object type, without
`@interfaceObject`. The exception does not apply, so the object type in source
schema B and the interface in source schema A are a genuine kind mismatch,
resulting in a `TYPE_KIND_MISMATCH` error.

```graphql counter-example
# Source Schema A: `Media` is an interface
interface Media {
  id: ID!
  title: String!
}

# Source Schema B: `Media` is a plain object type, not a stand-in
type Media {
  id: ID!
  reviewCount: Int!
}
```

### Validate Enums

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

### Validate Composite Types

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

- Let {fieldTypes} be the list of types of each field in {fields}.
- {LeastRestrictiveType(fieldTypes)} must not fail.

**Explanatory Text**

Fields on objects or interfaces that have the same name are considered
semantically equivalent and mergeable when {LeastRestrictiveType(fieldTypes)}
can select a return type for the composed field. This selection considers all
field types together and must not depend on source schema order.

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

Fields with leaf return types are not mergeable if the named types differ, or if
the same name is used with a different kind.

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

Fields with composite return types are mergeable when one of the declared return
types is a supertype of all other declared return types. The composed field uses
that supertype, regardless of the order in which the source schemas are
processed.

```graphql example
# Schema A
type Query @shareable {
  featured: FeaturedItem
}

union FeaturedItem = Product

type Product @shareable {
  id: ID
}

# Schema B
type Query @shareable {
  featured: Product
}

type Product @shareable {
  id: ID
}

# Composed Result
type Query {
  featured: FeaturedItem
}

union FeaturedItem = Product

type Product {
  id: ID
}
```

Fields with composite return types are not mergeable when no declared return
type is a supertype of all other declared return types.

```graphql counter-example
# Schema A
type Query @shareable {
  featured: FeaturedItem
}

union FeaturedItem = Product

type Product @shareable {
  id: ID
}

# Schema B
type Query @shareable {
  featured: Review
}

type Review @shareable {
  id: ID
}
```

#### Field Argument Types Mergeable

**Error Code**

`FIELD_ARGUMENT_TYPES_NOT_MERGEABLE`

**Severity**

ERROR

**Formal Specification**

- Let {typeNames} be the set of all output type names from all source schemas
  that are not declared as `@inaccessible` in any schema.
- For each {typeName} in {typeNames}
  - Let {types} be the set of all types with the {typeName} from all source
    schemas that are not declared as `@internal`.
  - Let {fieldNames} be the set of all field names from all {types} that are not
    declared as `@inaccessible` in any schema.
  - For each {fieldName} in {fieldNames}
    - Let {fields} be the set of all fields with the {fieldName} from all
      {types} that are not declared as `@internal`.
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
- {SameTypeShape(typeA, typeB)} must be true.

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

#### Field With Missing Required Arguments

**Error Code:**

`FIELD_WITH_MISSING_REQUIRED_ARGUMENT`

**Severity:**

ERROR

**Formal Specification:**

- Let {typeNames} be the set of all object and interface type names from all
  source schemas that are not declared as `@internal`
- For each {typeName} in {typeNames}:
  - Let {typeDefinitions} be the list of all type definitions from different
    source schemas with the name {typeName}.
  - Let {fieldNames} be the set of all field names from all {typeDefinitions}
    that are not declared as `@internal`.
  - For each {fieldName} in {fieldNames}:
    - Let {fieldDefinitions} be the list of all field definitions from
      {typeDefinitions} with the name {fieldName}.
    - Let {requiredArgumentNames} be the set of all argument names from
      {fieldDefinitions} that have a non-nullable type in at least one
      definition that does not specify `@require`
    - For each {fieldDefinition} in {fieldDefinitions}:
      - For each {requiredArgumentName} in {requiredArgumentNames}:
        - {fieldDefinition} must have an argument with the name
          {requiredArgumentName} that does not specify `@require`

**Explanatory Text:**

When merging a field definition across multiple schemas, any argument that is
non-null (i.e., “required”) in one schema must appear in all schemas that define
that field. In other words, arguments are effectively merged by intersection: if
an argument is considered required in any schema, then that same argument must
exist in every schema that contributes to the composite definition. If a
required argument is missing in one schema, there is no consistent way to define
that field across schemas.

If an argument is marked with `@require`, it is treated as non-required.
Consequently, this argument must either be nullable in all other schemas or must
also be marked with `@require` in all other schemas.

**Examples**

All schemas agree on having a required argument `author` for the `books` field:

```graphql example
# Schema A
type Query {
  books(author: String!): [Book] @shareable
}

# Schema B
type Query {
  books(author: String!): [Book] @shareable
}
```

In the following example, the `author` argument on the `books` field in Schema A
specifies a dependency on the `author` field in Schema C. The `author` argument
on the `books` field in Schema B is optional. As a result, the composition
succeeds; however, the `author` argument will not be included in the composite
schema.

```graphql example
# Schema A
type Collection {
  books(author: String! @require(field: "author")): [Book] @shareable
}

# Schema B
type Collection {
  books(author: String): [Book] @shareable
}

# Schema C
type Collection {
  author: String!
}
```

In the following counter-example, the `author` argument is required in one
schema but not in the other. This will result in a
`FIELD_WITH_MISSING_REQUIRED_ARGUMENT` error.

```graphql counter-example
# Schema A
type Query {
  books(author: String!): [Book] @shareable
}

# Schema B
type Query {
  books: [Book] @shareable
}
```

In the following counter-example, the `author` argument on the `books` field in
Schema A specifies a dependency on the `author` field in Schema C. The `author`
argument on the `books` field in Schema B is **not** optional. This will result
in a `FIELD_WITH_MISSING_REQUIRED_ARGUMENT` error.

```graphql counter-example
# Schema A
type Collection {
  books(author: String! @require(field: "author")): [Book] @shareable
}

# Schema B
type Collection {
  books(author: String!): [Book] @shareable
}

# Schema C
type Collection {
  author: String!
}
```

The same reasoning applies when the contributing schemas are `@interfaceObject`
stand-ins for the same interface, rather than ordinary object type declarations.
In the following counter-example, source schema B marks `minRating` with
`@require`. The executor supplies it from `Media.rating`, which is declared by
source schema A. Source schema C instead declares `minRating` as an ordinary,
non-nullable, client-supplied argument on the same field. In source schema B the
argument is executor-supplied; in source schema C it is client-supplied. The two
declarations are therefore not mergeable, and composition fails with a
`FIELD_WITH_MISSING_REQUIRED_ARGUMENT` error.

```graphql counter-example
# Source Schema A
interface Media @key(fields: "id") {
  id: ID!
  title: String!
  rating: Int!
}

# Source Schema B
type Media @interfaceObject @key(fields: "id") {
  id: ID!
  recommended(minRating: Int! @require(field: "rating")): [Review!]! @shareable
}

type Review {
  id: ID! @shareable
  rating: Int! @shareable
}

# Source Schema C
type Media @interfaceObject @key(fields: "id") {
  id: ID!
  recommended(minRating: Int!): [Review!]! @shareable
}

type Review {
  id: ID! @shareable
  rating: Int! @shareable
}
```

### Validate Input Types

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
    - {InputFieldsHaveConsistentDefaults(inputFields)} must be {true}.

InputFieldsHaveConsistentDefaults(inputFields):

- Given each pair of input fields {inputFieldA} and {inputFieldB} in
  {inputFields}:
  - If {inputFieldA} has a default value and {inputFieldB} has a default value:
    - If the default value of {inputFieldA} is not equal to the default value of
      {inputFieldB}:
      - return {false}
- return {true}

**Explanatory Text**

Input fields in different source schemas that have the same name are required to
have consistent default values. This ensures that there is no ambiguity or
inconsistency when merging input fields from different source schemas.

A mismatch in default values for input fields with the same name across
different source schemas will result in a schema composition error.

**Examples**

In the the following example both source schemas have an input field `genre`
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

If only one of the source schemas defines a default value for a given input
field, the composition is still valid:

```graphql example
# Schema A

input BookFilter {
  genre: Genre
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

### Validate External Directives

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
# Source Schema A
type Product {
  id: ID
  name: String
}

# Source Schema B
type Product {
  id: ID
  name: String @external
}
```

In this example, the `name` field on `Product` is marked as `@external` in
source schema B but has no non-`@external` declaration in any other source
schema, violating the rule:

```graphql counter-example
# Source Schema A
type Product {
  id: ID
}

# Source Schema B
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
# Source Schema A
type Product {
  name: String
}

# Source Schema B
type Product {
  name: String @external
}
```

In this example, the `@external` field `name` has a return type of `ProductName`
that doesn't match the base field's return type `String`, violating the rule:

```graphql counter-example
# Source Schema A
type Product {
  name: String
}

# Source Schema B
type Product {
  name: ProductName @external
}
```

### Validate Override Directives

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

`@override` is also legal on a field declared by an `@interfaceObject` stand-in.
It drops the field from the implementing types and stand-ins that the named
source schema declares. Because the named schema's own stand-in loses the field
as well, the projected implementation can move from one schema to another. A
projected field is not itself a source-schema declaration, so `@override` cannot
take it from an implementing type. A type that wants its own implementation
declares the field and marks it with `@implement` (see
[Interface Object Field Requires Implement](#sec-Interface-Object-Field-Requires-Implement)).
An `@override` that names a schema without a matching field drops nothing.

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

In this example, the implementation projected for `Media.reviews` moves from the
original "Reviews" schema to the new "Reviews2" schema. The "Reviews2" stand-in
overrides the "Reviews" stand-in field, so only the "Reviews2" declaration
supplies the projected implementation; no implementing type needs to change.

```graphql example
# The "Catalog" schema:
interface Media {
  id: ID!
  title: String!
}

# The original "Reviews" schema:
type Media @interfaceObject @key(fields: "id") {
  id: ID!
  reviews: [Review!]!
}

type Review {
  id: ID! @shareable
  rating: Int! @shareable
}

# The new "Reviews2" schema:
type Media @interfaceObject @key(fields: "id") {
  id: ID!
  reviews: [Review!]! @override(from: "Reviews")
}

type Review {
  id: ID! @shareable
  rating: Int! @shareable
}
```

### Validate Shareable Directives

#### Invalid Field Sharing

**Error Code**

`INVALID_FIELD_SHARING`

**Severity**

ERROR

**Formal Specification**

- Let {typeNames} be the set of all object type names from all source schemas
  that are not declared as `@internal`
- For each {typeName} in {typeNames}:
  - Let {typeDefinitions} be the list of all object type definitions from all
    source schemas with the name {typeName}.
  - Let {fieldNames} be the set of all field names from all {typeDefinitions}
    that are not declared as `@internal` or `@external`, part of a `@key`
    directive, or overridden.
  - For each {fieldName} in {fieldNames}:
    - Let {fieldDefinitions} be the list of all field definitions from
      {typeDefinitions} with the name {fieldName}.
      - If {fieldDefinitions} has more than one element:
        - For each {fieldDefinition} in {fieldDefinitions}:
          - {fieldDefinition} must be annotated with `@shareable`.

**Explanatory Text**

A field in a federated GraphQL schema may be marked `@shareable`, indicating
that the same field can be resolved by multiple schemas without conflict. When a
field is **not** marked as `@shareable` (sometimes called "non-shareable"), it
cannot be provided by more than one schema.

Field definitions marked as `@external` and overridden fields are excluded when
validating whether a field is shareable. These annotations indicate specific
cases where field ownership lies with another schema or has been replaced.

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
therefore the field can be defined in the other schema without being marked as
`@shareable`.

```graphql example
# Schema A
type User @key(fields: "id") {
  id: ID!
  fullName: String @override(from: "B")
}

# Schema B
type User @key(fields: "id") {
  id: ID!
  fullName: String
}
```

In the following example, `User.fullName` is marked as `@external` in one schema
and therefore the field can be defined in the other schema without being marked
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

In the following counter-example, `User.fullName` is non-shareable but is
defined and resolved by two different schemas, resulting in an
`INVALID_FIELD_SHARING` error.

```graphql counter-example
# Schema A
type User @key(fields: "id") {
  id: ID!
  fullName: String
}

# Schema B
type User @key(fields: "id") {
  id: ID!
  fullName: String
}
```

### Validate Interface Object Directives

#### Interface Object No Interface

**Error Code**

`INTERFACE_OBJECT_NO_INTERFACE`

**Severity**

ERROR

**Formal Specification**

- Let {schemas} be the set of all source schemas.
- Let {typeNames} be the set of all type names for which at least one schema in
  {schemas} declares an object type annotated with `@interfaceObject`.
- For each {typeName} in {typeNames}:
  - Let {definitions} be the set of all type definitions named {typeName} across
    {schemas}.
  - Let {interfaceDefinitions} be the subset of {definitions} that are interface
    types.
  - {interfaceDefinitions} must not be empty.

**Explanatory Text**

A stand-in binds to the interface of the same name. No source schema that
declares a stand-in references any other. At least one source schema **must**
define that name as an interface. If every source schema that declares the type
name uses `@interfaceObject`, and none defines it as an interface, the stand-in
has no interface to bind to and composition fails with an
`INTERFACE_OBJECT_NO_INTERFACE` error.

**Examples**

In this example, source schema A defines `Media` as a real interface, so source
schema B's stand-in is valid.

```graphql example
# Source Schema A
interface Media @key(fields: "id") {
  id: ID!
  title: String!
}

type Book implements Media {
  id: ID!
  title: String!
}

# Source Schema B
type Media @interfaceObject @key(fields: "id") {
  id: ID!
  reviews: [Review!]!
}

type Review {
  id: ID!
  rating: Int!
}
```

In the following counter-example, both source schema B and source schema C
declare `Media` as an `@interfaceObject` stand-in, but no source schema defines
`Media` as an interface. Neither stand-in has an interface to bind to, so
composition fails with an `INTERFACE_OBJECT_NO_INTERFACE` error.

```graphql counter-example
# Source Schema B
type Media @interfaceObject @key(fields: "id") {
  id: ID!
  reviews: [Review!]!
}

type Review {
  id: ID!
  rating: Int!
}

# Source Schema C
type Media @interfaceObject @key(fields: "id") {
  id: ID!
  averageRating: Float!
}
```

#### Interface Object Key Mismatch

**Error Code**

`INTERFACE_OBJECT_KEY_MISMATCH`

**Severity**

ERROR

**Formal Specification**

- Let {schemas} be the set of all source schemas.
- Let {standIns} be the set of all object types across {schemas} annotated with
  `@interfaceObject`.
- For each {standIn} in {standIns}:
  - Let {typeName} be the name of {standIn}.
  - Let {interfaceDefinitions} be the set of all interface types named
    {typeName} across {schemas}.
  - Let {interfaceKeys} be the set of field selection sets declared by `@key`
    directives on the types in {interfaceDefinitions}.
  - {interfaceKeys} must not be empty.
  - For each `@key` directive {keyDirective} on {standIn}:
    - Let {fieldSet} be the selection set of the `fields` argument of
      {keyDirective}.
    - {interfaceKeys} must contain an entry that selects the same fields as
      {fieldSet}.

**Explanatory Text**

An interface that has a stand-in must declare at least one key with `@key`. The
stand-in must key on one of the keys of the interface. Each `@key` on the
stand-in must select the same fields as a `@key` declared on the interface by at
least one interface-defining schema. The comparison is structural; the order of
the fields and their formatting do not matter.

**Examples**

In this example, the `Media` interface declares two keys, `id` and `sku`. The
stand-in in source schema B keys on `sku` alone. Because `sku` is one of the
keys of the interface, this is valid even though the stand-in does not repeat
the `id` key.

```graphql example
# Source Schema A
interface Media @key(fields: "id") @key(fields: "sku") {
  id: ID!
  sku: String!
  title: String!
}

# Source Schema B
type Media @interfaceObject @key(fields: "sku") {
  sku: String!
  reviews: [Review!]!
}

type Review {
  id: ID!
  rating: Int!
}
```

In the following counter-example, the stand-in in source schema B keys on `upc`,
but the `Media` interface declares no key with that field. This results in an
`INTERFACE_OBJECT_KEY_MISMATCH` error.

```graphql counter-example
# Source Schema A
interface Media @key(fields: "id") {
  id: ID!
  title: String!
}

# Source Schema B
type Media @interfaceObject @key(fields: "upc") {
  upc: String!
  reviews: [Review!]!
}

type Review {
  id: ID!
  rating: Int!
}
```

In the following counter-example, the `Media` interface declares no `@key` at
all. A stand-in for `Media` can therefore not declare a matching key, and
composition fails.

```graphql counter-example
# Source Schema A
interface Media {
  id: ID!
  title: String!
}

# Source Schema B
type Media @interfaceObject @key(fields: "id") {
  id: ID!
  reviews: [Review!]!
}

type Review {
  id: ID!
  rating: Int!
}
```

## Merge

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

### Merge Scalar Types

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

### Merge Interface Types

**Formal Specification**

MergeInterfaceTypes(types):

- If any {type} in {types} is marked with `@inaccessible`
  - Return {null}
- Let {firstType} be the first type in {types}.
- Let {typeName} be the name of {firstType}.
- Let {description} be the description of {firstType}.
- Let {mergedFields} be an empty set.
- For each {type} in {types}:
  - If {description} is {null}:
    - Set {description} to the description of {type}.
- Let {fieldNames} be the set of all field names in {types}.
- For each {fieldName} in {fieldNames}:
  - Let {fields} be the set of fields with the name {fieldName} in {types}.
  - Let {mergedField} be the result of {MergeOutputFields(fields)}.
  - If {mergedField} is not {null}:
    - Add {mergedField} to {mergedFields}.
- Return a new interface type with the name of {typeName}, description of
  {description}, and fields of {mergedFields}.

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
across multiple interfaces are reconciled via {MergeOutputFields(fields)}. This
ensures any differences in type, nullability, or other constraints are resolved
before appearing in the final interface.

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

interface Product {
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

### Merge Enum Types

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
- Let {description} be the first non empty description of any {enum} in {enums}.
- Let {mergedValues} be an empty set.
- Let {valueNames} be the set of all enum value names in {enums}.
- For each {valueName} in {valueNames}:
  - Let {values} be the set of enum values with the name {valueName} in {enums}.
  - Let {mergedValue} be the result of {MergeEnumValues(values)}.
  - If {mergedValue} is not {null}:
    - Add {mergedValue} to {mergedValues}.
- Return a new enum type with the name of {typeName}, description of
  {description}, and enum values of {mergedValues}.

MergeEnumValues(enumValues):

- If any {enumValue} in {enumValues} is marked with `@inaccessible`
  - Return {null}
- Let {name} be the name of the first {enumValue} in {enumValues}.
- Let {description} be the first non empty description of any {enumValue} in
  {enumValues}.
- Return a new enum value with the name of {name} and description of
  {description}.

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

### Merge Union Types

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

### Merge Input Types

**Formal Specification**

MergeInputTypes(types):

- If any {type} in {types} is marked with `@inaccessible`
  - Return {null}
- Let {firstType} be the first type in {types}.
- Let {typeName} be the name of {firstType}.
- Let {description} be the description of {firstType}.
- Let {mergedFields} be an empty set.
- For each {type} in {types}:
  - If {description} is {null}:
    - Set {description} to the description of {type}.
- Let {fieldNames} be the set of all field names in {types}.
- For each {fieldName} in {fieldNames}:
  - Let {fieldDefinitions} be the set of fields with the name {fieldName} in
    {types}.
  - If length of {fieldDefinitions} is not equal to the length of {types}:
    - Continue
  - If any field in {fieldDefinitions} is marked with `@inaccessible`
    - Continue
  - Let {mergedField} be the result of {MergeInputFields(fieldDefinitions)}.
  - If {mergedField} is not {null}:
    - Add {mergedField} to {fields}.
- If {fields} is empty:
  - Return {null}
- Return a new input type with the name of {typeName}, description of
  {description}, and fields of {fields}.

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
name found across the remaining types. For each field,
{MergeInputFields(fields)} is called to reconcile differences in type,
nullability, default values, etc.. If a merged field ends up being `null` - for
instance, because one of its underlying definitions was inaccessible - that
field is not included in the final definition. The end result is a single input
type that correctly unifies every compatible field from the various sources.

After filtering out inaccessible types, the algorithm takes the **intersection**
of the field names across the remaining types - only those fields that appear in
**every** source definition are eligible. For each eligible field, it invokes
{MergeInputFields(fieldsForName)} to reconcile differences in type, nullability,
default values, etc. The end result is a single input type that correctly
unifies every compatible field that appears in all source types.

**Examples**

Here, two `OrderInput` input types from different schemas are merged into a
single composed `OrderInput` type. Notice that only the fields present in _both_
schemas are included.

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
}
```

Although `description` appears in Schema A and `total` appears in Schema B,
neither field is defined in _both_ schemas; therefore, only `id` remains.

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

### Merge Object Types

**Formal Specification**

MergeObjectTypes(types):

- If any {type} in {types} is marked with `@inaccessible`
  - Return {null}
- Remove all types marked with `@internal` from {types}.
- If {types} is empty:
  - Return {null}
- Let {firstType} be the first type in {types}.
- Let {typeName} be the name of {firstType}.
- Let {description} be the description of {firstType}.
- Let {mergedFields} be an empty set.
- For each {type} in {types}:
  - If {description} is {null}:
    - Set {description} to the description of {type}.
- Let {fieldNames} be the set of all field names in {types}.
- For each {fieldName} in {fieldNames}:
  - Let {fields} be the set of fields with the name {fieldName} in {types}.
  - Let {mergedField} be the result of {MergeOutputFields(fields)}.
  - If {mergedField} is not {null}:
    - Add {mergedField} to {mergedFields}.
- Return a new object type with the name of {typeName}, description of
  {description}, fields of {mergedFields}.

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
field name across these types, then calls {MergeOutputFields(fields)} for each
name to reconcile any differences. If {MergeOutputFields(fields)} returns {null}
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

### Merge Interface Implementations

**Formal Specification**

MergeInterfaceImplementations(schemas):

- Let {edges} be an empty set of ({type}, {interface}) pairs.
- For each {schema} in {schemas}:
  - Let {typeDefinitions} be the set of all object and interface type
    definitions in {schema}.
  - For each {typeDefinition} in {typeDefinitions}:
    - Let {typeName} be the name of {typeDefinition}.
    - Let {interfaceNames} be the set of names of the interfaces that
      {typeDefinition} declares as implemented.
    - For each {interfaceName} in {interfaceNames}:
      - Add the pair ({typeName}, {interfaceName}) to {edges}.
- Return the result of {CloseImplementsEdges(edges)}.

CloseImplementsEdges(edges):

Completes {edges} so that implementation is transitive: whenever {type}
implements {interface}, and {interface} itself implements {parentInterface}, the
pair ({type}, {parentInterface}) is added to the result.

- Let {closedEdges} be a copy of {edges}.
- Let {worklist} be a copy of {edges}.
- While {worklist} is not empty:
  - Remove one pair ({type}, {interface}) from {worklist}.
  - For each pair ({interface}, {parentInterface}) in {closedEdges}:
    - If the pair ({type}, {parentInterface}) is not in {closedEdges}:
      - Add the pair ({type}, {parentInterface}) to {closedEdges}.
      - Add the pair ({type}, {parentInterface}) to {worklist}.
- Return {closedEdges}.

**Explanatory Text**

{MergeInterfaceImplementations(schemas)} computes the complete `implements`
relation for the composite schema: which object and interface types implement
which interfaces. It combines every source schema's local declarations and
completes the result so that implementation is always transitive.
{MergeObjectTypes(types)} and {MergeInterfaceTypes(types)} use its result to
construct each merged type's `implements` clause. Every post-merge rule that
reasons about interface implementation uses it too.

_Combining Declared Implementations_

Every `implements` relationship declared on an object or interface type in any
source schema contributes one pair to {edges}. A type need not declare the same
interfaces consistently across every source schema that defines it. The pair
that any single schema contributes is enough to make that implementation part of
the composite schema. This mirrors how fields and descriptions are combined
elsewhere during merging. Each source schema contributes a partial view, and
composition unions them.

_Completing the Hierarchy_

The GraphQL specification requires that a type transitively implement every
interface implemented by any interface it implements. For example, if
`PhysicalProduct` implements `Product`, every type that implements
`PhysicalProduct` must also declare that it implements `Product`. Each source
schema declares only its local portion of the hierarchy. This obligation
therefore often spans schema boundaries that no individual source schema can
satisfy on its own. {CloseImplementsEdges(edges)} closes the relation over this
rule. It adds the missing edges automatically, so the composite schema, taken as
a whole, stays valid GraphQL.

_Partial Views Are Expected_

A source schema that declares `PhysicalProduct implements Product` need not
reference every other interface layered onto the same hierarchy elsewhere. A
source schema that declares `Chair implements PhysicalProduct` need not define
`Product`. Neither schema is incomplete or incorrect on its own; only the
composite schema must reflect the full hierarchy. Partial, schema-local views of
a shared interface hierarchy are expected.
{MergeInterfaceImplementations(schemas)} reconciles them into a single,
transitively closed relation.

_Ordering Within Composition_

{MergeInterfaceImplementations(schemas)} runs before {MergeObjectTypes(types)},
{MergeInterfaceTypes(types)}, and before any post-merge validation. Field
merging depends on it because the `implements` clause of a merged type is
produced together with its merged fields, as part of the same type.

Post-merge rules such as `INTERFACE_FIELD_NO_IMPLEMENTATION` and
`IMPLEMENTED_BY_INACCESSIBLE` depend on it because both are defined in terms of
"the set of interfaces implemented by {type}" in the merged schema. Neither rule
changes to accommodate this algorithm. They evaluate against the complete
relation instead of whatever subset of it a single source schema happened to
declare. They continue to gate contract completeness exactly as before. A field
that a type carries only to satisfy an interface reached through closure is held
to the same standard as one reached through a direct declaration.

Note: Source schemas hold only partial views of the hierarchy. The distributed
executor must not assume that a value's originating source schema defines every
interface the composite schema records for that value. The distributed executor
resolves abstract-type membership against the composite schema. This includes,
for example, which concrete type backs a value, for `__typename` or a type
condition. The executor rewrites any type condition it sends to a source schema
into that schema's own local type vocabulary. It never sends a source schema a
type condition naming a type that source schema does not define.

Note: The union alone can produce a schema that violates the GraphQL
specification's transitive implementation rule. This can happen because a type
may inherit an interface only through an intermediate interface defined in a
different source schema. Completing the transitive closure automatically fixes
this. It lets source schemas with correct but partial views of a shared
hierarchy compose successfully, without any one of them needing full knowledge
of it.

**Examples**

In this example, two source schemas each declare one interface on the shared
`Chair` type. Neither interface is related to the other, so composition unions
the two declarations.

```graphql example
# Source Schema A
interface Product {
  id: ID!
}

type Chair implements Product @key(fields: "id") {
  id: ID!
  legs: Int
}

# Source Schema B
interface Searchable {
  score: Float
}

type Chair implements Searchable @key(fields: "id") {
  id: ID!
  score: Float
}

# Composite Schema
interface Product {
  id: ID!
}

interface Searchable {
  score: Float
}

type Chair implements Product & Searchable {
  id: ID!
  legs: Int
  score: Float
}
```

In the following example, source schema A declares that `PhysicalProduct`
implements `Product`, while source schema B declares `PhysicalProduct` again,
without that relationship, and separately declares that `Chair` implements
`PhysicalProduct`. Source schema B never mentions `Product`.

```graphql example
# Source Schema A
interface Product {
  id: ID!
}

interface PhysicalProduct implements Product {
  id: ID!
  weight: Int
}

# Source Schema B
interface PhysicalProduct {
  id: ID!
  weight: Int
}

type Chair implements PhysicalProduct {
  id: ID!
  weight: Int
  legs: Int
}

# Composite Schema
interface Product {
  id: ID!
}

interface PhysicalProduct implements Product {
  id: ID!
  weight: Int
}

type Chair implements PhysicalProduct & Product {
  id: ID!
  weight: Int
  legs: Int
}
```

Composing `PhysicalProduct` contributes the pair (`PhysicalProduct`, `Product`),
declared directly by source schema A. Composing `Chair` contributes the pair
(`Chair`, `PhysicalProduct`), declared directly by source schema B.
{CloseImplementsEdges(edges)} then finds that `Chair` implements
`PhysicalProduct`, and `PhysicalProduct` implements `Product`, and adds the pair
(`Chair`, `Product`) to the result even though no source schema declared it.
This derived edge lets the composed `Chair` type explicitly implement `Product`,
as GraphQL requires. It is recorded as derived, rather than declared, so that
tooling can explain why `Chair` implements an interface that no single source
schema named.

### Project Interface Object Fields

**Formal Specification**

ProjectInterfaceObjectFields(schemas, mergedSchema):

- Assert: The implements relation of {mergedSchema} is complete (see
  [Merge Interface Implementations](#sec-Merge-Interface-Implementations)).
- Let {interfaceTypes} be the set of all interface types in {mergedSchema} for
  which at least one stand-in exists, where a stand-in for an interface is an
  object type annotated with `@interfaceObject` that has the same name as the
  interface.
- For each {interface} in {interfaceTypes}:
  - Let {drops} be the result of {CollectStandInOverrides(interface, schemas,
    mergedSchema)}.
  - Remove each field declaration in {drops} from the type that declares it.
- For each {interface} in {interfaceTypes}:
  - Let {contractFields} be the result of {MergeContractFields(interface,
    schemas)}.
  - Set the fields of {interface} to {contractFields}.
- For each {interface} in {interfaceTypes}:
  - Let {contributedFields} be the result of {ContributedFields(interface,
    schemas)}.
  - For each interface type {subInterface} in {mergedSchema} that implements
    {interface}:
    - For each {contributedField} in {contributedFields}:
      - Let {fieldName} be the name of {contributedField}.
      - If {subInterface} has no field named {fieldName}:
        - Add {contributedField} to the fields of {subInterface}.
      - Otherwise:
        - Let {ownDeclarations} be the fields named {fieldName} declared on a
          stand-in for {subInterface}.
        - Let {lessSpecificDeclarations} be the fields named {fieldName}
          declared on a stand-in for {interface}.
        - If every declaration in {ownDeclarations} is annotated with
          `@implement`:
          - Continue to the next {contributedField}.
        - If no declaration in {ownDeclarations} is annotated with `@implement`
          and every declaration in {ownDeclarations} and
          {lessSpecificDeclarations} is annotated with `@shareable` or is part
          of a `@key` directive on its stand-in:
          - Continue to the next {contributedField}.
        - For each declaration in {ownDeclarations} that is not annotated with
          `@implement`:
          - Record that {declaration} collides with a projected implementation
            without declaring either replacement or sharing.
- For each object type {objectType} in {mergedSchema}:
  - Let {implementedInterfaces} be the set of all interfaces in {interfaceTypes}
    that {objectType} implements in {mergedSchema}.
  - Let {projectedFieldNames} be an empty set.
  - For each {interface} in {implementedInterfaces}:
    - Add the name of each field in {ContributedFields(interface, schemas)} to
      {projectedFieldNames}.
  - For each {fieldName} in {projectedFieldNames}:
    - Let {candidates} be the result of {ContributingDeclarations(objectType,
      fieldName, schemas, mergedSchema)}.
    - Assert: {candidates} is not empty.
    - Let {projectedDeclarations} be the result of
      {EffectiveContributingDeclarations(candidates, mergedSchema)}.
    - Let {directDeclarations} be the set of all fields named {fieldName}
      declared on object types named the name of {objectType} across {schemas},
      excluding declarations annotated with `@external` or `@internal` and
      declarations overridden through an `@override` directive in another source
      schema.
    - If {directDeclarations} is not empty:
      - If every declaration in {directDeclarations} is annotated with
        `@implement`:
        - Let {ownerDeclarations} be {directDeclarations}.
      - Otherwise, if no declaration in {directDeclarations} is annotated with
        `@implement` and every declaration in {directDeclarations} and
        {projectedDeclarations} is annotated with `@shareable` or is part of a
        `@key` directive on its declaring type:
        - Let {ownerDeclarations} be the union of {directDeclarations} and
          {projectedDeclarations}.
      - Otherwise:
        - Let {ownerDeclarations} be {directDeclarations}.
        - For each declaration in {directDeclarations} that is not annotated
          with `@implement`:
          - Record that {declaration} collides with a projected implementation
            without declaring either replacement or sharing.
    - Otherwise:
      - Let {ownerDeclarations} be {projectedDeclarations}.
    - Let {minimalProjectedDeclarations} be the subset of
      {projectedDeclarations} for which no other declaration in
      {projectedDeclarations} has a different contributing interface that
      implements the contributing interface of that declaration in
      {mergedSchema}.
    - If {directDeclarations} is empty, {minimalProjectedDeclarations} contains
      more than one declaration, and not every declaration in
      {minimalProjectedDeclarations} is annotated with `@shareable` or is part
      of a `@key` directive on its stand-in:
      - Record a projected field sharing conflict for ({objectType},
        {fieldName}) and the declarations in {minimalProjectedDeclarations}.
    - Record ({objectType}, {fieldName}) as owned by the schemas defining
      {ownerDeclarations}. For each projected owner, also record its
      contributing interface and, for each implements edge through which
      {objectType} implements that interface in {mergedSchema}, whether that
      edge was declared in a source schema or derived.
    - If {ownerDeclarations} contains both direct and projected declarations:
      - Let {mergedField} be the result of
        {MergeOutputFields(ownerDeclarations)}.
      - Set the field named {fieldName} on {objectType} to {mergedField}.
    - Otherwise, if {objectType} has no field named {fieldName}:
      - Let {ownerInterface} be the contributing interface of the first
        declaration in {ownerDeclarations}.
      - Add the field named {fieldName} of {ownerInterface} to the fields of
        {objectType}.

CollectStandInOverrides(interface, schemas, mergedSchema):

- Let {drops} be an empty set.
- For each stand-in {standIn} for {interface} across {schemas}:
  - For each field {field} on {standIn} annotated with `@override`:
    - Let {fieldName} be the name of {field}.
    - Let {from} be the value of the `from` argument of the `@override`
      directive on {field}.
    - Let {sourceSchema} be the source schema named {from} in {schemas}.
    - If {sourceSchema} does not exist or {sourceSchema} is the schema defining
      {standIn}:
      - Continue.
    - Let {closureTypes} be the set containing {interface} and every type in
      {mergedSchema} that implements {interface}.
    - For each {closureType} in {closureTypes}:
      - Let {localType} be the type named the name of {closureType} defined in
        {sourceSchema}, or {null} if {sourceSchema} defines no such type.
      - If {localType} is not {null} and {localType} is an object type and
        {localType} declares a field named {fieldName}:
        - Add that field declaration to {drops}.
- Return {drops}.

MergeContractFields(interface, schemas):

- Let {interfaceName} be the name of {interface}.
- Let {declarations} be the set of all fields declared on any interface type
  named {interfaceName} across {schemas} or on any stand-in for {interface}.
- Let {fieldNames} be the set of the names of all fields in {declarations}.
- Let {contractFields} be an empty set.
- For each {fieldName} in {fieldNames}:
  - Let {fields} be the set of all fields named {fieldName} in {declarations}.
  - Let {mergedField} be the result of {MergeOutputFields(fields)}.
  - If {mergedField} is not {null}:
    - Add {mergedField} to {contractFields}.
- Return {contractFields}.

ContributedFields(interface, schemas):

- Let {contractFields} be the result of {MergeContractFields(interface,
  schemas)}.
- Let {contributedFields} be an empty set.
- For each {contractField} in {contractFields}:
  - If any stand-in for {interface} declares a field named the name of
    {contractField} that is not annotated with `@internal` and that is not
    selected by any `@key` directive on that stand-in:
    - Add {contractField} to {contributedFields}.
- Return {contributedFields}.

ContributingDeclarations(objectType, fieldName, schemas, mergedSchema):

- Let {candidates} be an empty set.
- For each interface type {interface} in {mergedSchema} that {objectType}
  implements:
  - For each stand-in {standIn} for {interface} across {schemas}:
    - For each field {declaration} named {fieldName} on {standIn} that is not
      annotated with `@internal`:
      - Add {declaration}, with {interface} as its contributing interface, to
        {candidates}.
- Return {candidates}.

EffectiveContributingDeclarations(candidates, mergedSchema):

- Let {effectiveDeclarations} be a copy of {candidates}.
- Let {contributingInterfaces} be the set of contributing interfaces represented
  in {candidates}.
- For each {contributingInterface} in {contributingInterfaces}:
  - Let {ownDeclarations} be the declarations in {candidates} whose contributing
    interface is {contributingInterface}.
  - If {ownDeclarations} is not empty and every declaration in {ownDeclarations}
    is annotated with `@implement`:
    - Remove from {effectiveDeclarations} every declaration whose contributing
      interface is implemented, directly or transitively, by
      {contributingInterface}.
- Return {effectiveDeclarations}.

**Explanatory Text**

{ProjectInterfaceObjectFields(schemas, mergedSchema)} does two things. It adds
the fields that stand-ins contribute to the composed interface contracts. For
every implementing object type, it also decides which source schema provides
each contributed field. Composition makes this decision statically. For every
pair of type and field, the composed schema records the set of source schemas
that the _distributed executor_ may use. The set has one member for an
exclusively owned field and may have several members for a shareable field. No
undeclared ambiguity remains at runtime.

_Stand-Ins and Preconditions_

A _stand-in_, as defined by `@interfaceObject` in Section 2 -- Source Schema, is
an object type that shares the name of an interface defined by at least one
other source schema. Composition projects its non-key field implementations onto
the interface's implementing types. A stand-in never appears in the composed
schema. Every reference to it becomes a reference to the interface.

This algorithm runs after two earlier steps. First,
[Merge Interface Implementations](#sec-Merge-Interface-Implementations)
completes the implements relation of the merged schema, so both declared and
derived implements edges are available. Second, type groups have already been
formed, so each stand-in has been merged into the type group of its interface
rather than treated as a kind conflict.

_Applying Overrides_

A stand-in field may be annotated with `@override(from: ...)`. Before
composition resolves precedence, {CollectStandInOverrides(interface, schemas,
mergedSchema)} removes every declaration of that field in the named source
schema. This applies to declarations on implementing object types, on the
stand-ins of more specific interfaces, and on that schema's own stand-in for the
interface. All of these are object types. A field declared on an interface
itself is never removed. Removing the source schema's own stand-in declaration
lets the projected implementation move from one schema to another.

A projected field is not a declaration for `@override` purposes. An implementing
type cannot use `@override` to replace a projected implementation; `@implement`
does that instead. A type that acquires a field through `@override` counts as a
direct declarer for the precedence resolution below. It still requires
`@implement` when a projected implementation applies. Overrides that name a
nonexistent schema or the declaring schema itself, and override cycles, are
covered by the existing `@override` validation rules. This includes
`OVERRIDE_FROM_SELF` and `OVERRIDE_SOURCE_HAS_OVERRIDE`, which apply unchanged.

_Extending the Interface Contract_

{MergeContractFields(interface, schemas)} merges stand-in field declarations
into the interface's field set alongside the interface-declared fields. It uses
the same {MergeOutputFields(fields)} machinery applied everywhere else (see
[Merge Output Fields](#sec-Merge-Output-Fields)). No new merge rules are
introduced. Stand-in key fields are part of a `@key` directive, so they are
already implicitly shareable. They merge with same-named interface fields
without triggering sharing conflicts. This is an inherited property of key
fields, not an exception introduced here. A stand-in field that is part of a
`@key` merges into the contract but is never projected as a field
implementation.

Arguments of contributed fields that are annotated with `@require` are excluded
from the composed schema by the existing behavior of
{MergeOutputFields(fields)}. Their values are supplied by the distributed
executor instead. This algorithm requires no additional step to strip them.

_Propagating Contributed Fields_

Every contributed field propagates into the contract of every sub-interface of
the contributing interface. This uses the completed implements relation. A
sub-interface that already declares the field keeps its own declaration. A real
interface field is only a contract declaration; it provides no implementation. A
more-specific stand-in's own declaration may replace an implementation projected
from a less-specific interface by using `@implement`, or may coexist with it
when every applicable declaration is `@shareable`. The
[GraphQL specification](https://spec.graphql.org/) already governs how field
types and argument rules must stay compatible between a contract and the fields
of implementing types. This specification adds no further compatibility rule.

_Resolving Effective Owners_

For each implementing object type and each contributed field name, composition
resolves an effective owner set. `@implement` and `@shareable` affect different
parts of this resolution.

`@implement` controls precedence. When every declaration contributed at a more
specific interface is annotated with `@implement`, declarations contributed by
the less-specific interfaces that it implements are removed from the owner set.
Likewise, when every direct declaration on an implementing object type is
annotated with `@implement`, all applicable projected implementations are
removed. Direct declarations in several source schemas remain together at the
same specificity level; `@implement` does not choose between them.

`@shareable` controls multiplicity. When no direct declaration uses
`@implement`, the direct and projected declarations may remain together only
when every one of them is shareable. All of their schemas are then effective
owners, and the distributed executor may use any reachable owner. The same rule
allows implementations projected from more-specific and less-specific stand-ins
to coexist when neither uses `@implement`. The declarations participate together
in {MergeOutputFields(fields)}, so their field types and arguments must be
mergeable.

Direct declarations in multiple schemas continue to use the ordinary sharing
rule, including `INVALID_FIELD_SHARING`. Several incomparable projected
implementations continue to use `INVALID_PROJECTED_FIELD_SHARING`. Key fields
never conflict here because they are already implicitly shareable.

_Explicit Replacement_

Replacing a projected implementation is always explicit. A direct declaration on
an implementing type, or a more-specific stand-in declaration, must use
`@implement` to remove an applicable projected implementation from the owner
set. It may omit `@implement` only when all colliding declarations are
`@shareable`; that choice keeps both implementations eligible. Any other
unmarked collision is reported by the _Post Merge Validation_ rule
`INTERFACE_OBJECT_FIELD_REQUIRES_IMPLEMENT`.

When multiple source schemas directly declare the replacing field, every
declaration uses `@implement` to express the same precedence decision. If those
declarations are all to remain eligible, they also use `@shareable`.
`@implement` therefore does not imply exclusive ownership; it removes
less-specific projected implementations, while `@shareable` permits multiple
owners among the declarations that remain.

The inverse condition, an `@implement` annotation with no applicable projected
implementation, is reported by `IMPLEMENT_WITHOUT_PROJECTED_IMPLEMENTATION`.
Together these rules guarantee that adding a stand-in field never silently
changes the behavior of any implementing type, and that replacing its projected
implementation is always visible in the source text of every schema that
replaces it.

_Ownership Metadata_

For every pair of object type and contributed field, the algorithm records the
effective owner set. For each projected owner it records the contributing
interface. For each implements edge through which the object type reaches that
interface, it also records whether the edge was declared in a source schema or
derived during
[Merge Interface Implementations](#sec-Merge-Interface-Implementations). The
_Validate Satisfiability_ rules use this information. The distributed executor
uses it too, to send each field to any reachable effective owner. The
derived-edge record lets diagnostics explain why an implementation is projected
onto a type whose source schema never declared the corresponding implements
edge.

**Examples**

In the following example, source schema A defines the `Media` interface with its
implementing types `Book` and `Movie`. Source schema B contributes a `reviews`
field to `Media` through a stand-in. Source schema C replaces the implementation
projected onto `Photo` with its own implementation, marked with `@implement`.

```graphql example
# Source Schema A
interface Media @key(fields: "id") {
  id: ID!
  title: String!
}

type Book implements Media {
  id: ID!
  title: String!
}

type Movie implements Media {
  id: ID!
  title: String!
}

# Source Schema B
type Media @interfaceObject @key(fields: "id") {
  id: ID!
  reviews: [Review!]!
}

type Review {
  body: String! @shareable
}

# Source Schema C
interface Media {
  id: ID!
  title: String!
}

type Photo implements Media {
  id: ID!
  title: String!
  reviews: [Review!]! @implement
}

type Review {
  body: String! @shareable
}

# Composite Schema
interface Media {
  id: ID!
  title: String!
  reviews: [Review!]!
}

type Book implements Media {
  id: ID!
  title: String!
  # Projected implementation; effective owner: source schema B
  reviews: [Review!]!
}

type Movie implements Media {
  id: ID!
  title: String!
  # Projected implementation; effective owner: source schema B
  reviews: [Review!]!
}

type Photo implements Media {
  id: ID!
  title: String!
  # Own implementation; effective owner: source schema C
  reviews: [Review!]!
}

type Review {
  body: String!
}
```

The stand-in's `reviews` field merges into the `Media` contract and is projected
into every implementing type. `Book` and `Movie` declare no `reviews` field of
their own, so source schema B's implementation is projected onto them. Their
effective owner for `reviews` is source schema B, and source schema A needs no
change. `Photo` declares its own `reviews` field with `@implement`, so the
projected implementation is removed from its owner set and its effective owner
is source schema C. Omitting `@implement` would be reported by
`INTERFACE_OBJECT_FIELD_REQUIRES_IMPLEMENT` because the declarations are not
both `@shareable`.

The stand-in's `id` field is part of its `@key` directive, so it is already
implicitly shareable. It merges with the `id` declarations of the interface and
its implementing types without requiring `@implement` or `@shareable`. The
stand-in itself does not appear in the composed schema. Every reference to it is
a reference to the `Media` interface.

The following examples show how `@implement` and `@shareable` independently
affect the effective owner set of a projected field.

In this example, source schema B supplies the only implementation of `taxRate`.
The field is projected onto both `Chair` and `Table`.

```graphql example
# Source Schema A
type Query {
  productById(id: ID!): Product @lookup
}

interface Product @key(fields: "id") {
  id: ID!
  name: String!
}

type Chair implements Product @key(fields: "id") {
  id: ID!
  name: String!
}

type Table implements Product @key(fields: "id") {
  id: ID!
  name: String!
}

# Source Schema B
type Query {
  productTaxById(id: ID!): Product @lookup @internal
}

type Product @interfaceObject @key(fields: "id") {
  id: ID!
  taxRate: Float
}
```

The effective owner of both `Chair.taxRate` and `Table.taxRate` is source schema
B.

In the next example, source schema B's projected implementation and source
schema C's direct implementation are both `@shareable`. Neither declaration uses
`@implement`, so neither replaces the other.

```graphql example
# Source Schema A
type Query {
  productById(id: ID!): Product @lookup
}

interface Product @key(fields: "id") {
  id: ID!
  name: String!
}

type Chair implements Product @key(fields: "id") {
  id: ID!
  name: String!
}

type Table implements Product @key(fields: "id") {
  id: ID!
  name: String!
}

# Source Schema B
type Query {
  productTaxById(id: ID!): Product @lookup @internal
}

type Product @interfaceObject @key(fields: "id") {
  id: ID!
  taxRate: Float @shareable
}

# Source Schema C
type Query {
  chairTaxById(id: ID!): Chair @lookup @internal
}

type Chair @key(fields: "id") {
  id: ID!
  taxRate: Float @shareable
}
```

The effective owners of `Chair.taxRate` are source schemas B and C. The
distributed executor may use either reachable implementation. The effective
owner of `Table.taxRate` remains source schema B.

In the next example, source schema C marks its direct declaration with
`@implement`. This removes the less-specific projected implementation from the
owner set for `Chair`, without affecting other implementations of `Product`.

```graphql example
# Source Schema A
type Query {
  productById(id: ID!): Product @lookup
}

interface Product @key(fields: "id") {
  id: ID!
  name: String!
}

type Chair implements Product @key(fields: "id") {
  id: ID!
  name: String!
}

type Table implements Product @key(fields: "id") {
  id: ID!
  name: String!
}

# Source Schema B
type Query {
  productTaxById(id: ID!): Product @lookup @internal
}

type Product @interfaceObject @key(fields: "id") {
  id: ID!
  taxRate: Float @shareable
}

# Source Schema C
type Query {
  chairTaxById(id: ID!): Chair @lookup @internal
}

type Chair @key(fields: "id") {
  id: ID!
  taxRate: Float @implement
}
```

The effective owner of `Chair.taxRate` is source schema C. The effective owner
of `Table.taxRate` remains source schema B.

Finally, two source schemas provide the explicit implementation for `Chair`.
Both declarations use `@implement` to replace the less-specific projected
implementation and `@shareable` to remain interchangeable with each other.

```graphql example
# Source Schema A
type Query {
  productById(id: ID!): Product @lookup
}

interface Product @key(fields: "id") {
  id: ID!
  name: String!
}

type Chair implements Product @key(fields: "id") {
  id: ID!
  name: String!
}

type Table implements Product @key(fields: "id") {
  id: ID!
  name: String!
}

# Source Schema B
type Query {
  productTaxById(id: ID!): Product @lookup @internal
}

type Product @interfaceObject @key(fields: "id") {
  id: ID!
  taxRate: Float @shareable
}

# Source Schema C
type Query {
  chairTaxById(id: ID!): Chair @lookup @internal
}

type Chair @key(fields: "id") {
  id: ID!
  taxRate: Float @implement @shareable
}

# Source Schema D
type Query {
  chairTaxByLegacyId(id: ID!): Chair @lookup @internal
}

type Chair @key(fields: "id") {
  id: ID!
  taxRate: Float @implement @shareable
}
```

The effective owners of `Chair.taxRate` are source schemas C and D. Source
schema B is not an effective owner for `Chair.taxRate`, because the explicit
implementations replace its less-specific projected implementation. Source
schema B remains the effective owner of `Table.taxRate`.

### Merge Output Fields

**Formal Specification**

MergeOutputFields(fields):

- If any {field} in {fields} is marked with `@inaccessible`
  - Return {null}
- Filter out all fields marked with `@internal` from {fields}.
- If {fields} is empty:
  - Return {null}
- Let {firstField} be the first field in {fields}.
- Let {fieldName} be the name of {firstField}.
- Let {fieldTypes} be the list of types of each field in {fields}.
- Let {fieldType} be the result of {LeastRestrictiveType(fieldTypes)}.
- Let {description} be the description of {firstField}.
- For each {field} in {fields}:
  - If {description} is {null}:
    - Let {description} be the description of {field}.
- Let {mergedArguments} be an empty set.
- Let {argumentNames} be the set of all argument names in {fields}.
- For each {argumentName} in {argumentNames}:
  - Let {arguments} be the set of arguments with the name {argumentName} in
    {fields}
  - If length of {arguments} is not equal to the length of {fields}:
    - Continue.
  - If any argument in {arguments} is marked with `@inaccessible`:
    - Continue.
  - If any argument in {arguments} is marked with `@require`:
    - Continue.
  - Let {mergedArgument} be the result of {MergeArgumentDefinitions(arguments)}.
  - If {mergedArgument} is not {null}:
    - Add {mergedArgument} to {mergedArguments}.
- Return a new field with the name of {fieldName}, type of {fieldType},
  arguments of {mergedArguments}, and description of {description}.

**Explanatory Text**

The {MergeOutputFields(fields)} algorithm is used when multiple fields across
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
{LeastRestrictiveType(fieldTypes)} with the complete list of field return types.
This helper function computes a type that is compatible with all the provided
field types, ensuring that the composed schema does not break schemas expecting
any of those types. The calculation is order-independent. For example,
{LeastRestrictiveType(fieldTypes)} might unify `String!` and `String` into
`String`, or `A` and `U` into `U` when `U` is a union that contains `A`.

_Merging Arguments_

Each field can declare arguments. The algorithm collects all argument names
across these fields and merges them using {MergeArgumentDefinitions(arguments)}.
Before merging, any arguments marked with `@inaccessible` or `@require` are
excluded. If this exclusion causes the number of arguments available for a given
name to differ from the total number of fields - or if at least one schema omits
the argument - the argument is skipped entirely. Otherwise, any differences in
argument type, default value, or description are resolved via the merging rules
in {MergeArgumentDefinitions(arguments)}.

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

If the argument is missing in one of the schemas, the composed field will not
include that argument:

```graphql example
# Schema A
type Product {
  discountPercentage(percent: Int): Int
}

# Schema B
type Product {
  discountPercentage: Int
}

# Composed Result
type Product {
  discountPercentage: Int
}
```

In case one argument is marked with `@inaccessible`, the composed field will not
include that argument:

```graphql example
# Schema A
type Product {
  discountPercentage(percent: Int): Int
}

# Schema B
type Product {
  discountPercentage(percent: Int @inaccessible): Int
}

# Composed Result
type Product {
  discountPercentage: Int
}
```

In case a schema defines a requirement through the `@require` directive, the
composed field will not include that argument

```graphql example
# Schema A
type Product {
  discountPercentage(percent: Int): Int
  discount: Int
}

# Schema B
type Product {
  discountPercentage(percent: Int @require(field: "discount")): Int
}

# Composed Result
type Product {
  discountPercentage: Int
}
```

### Merge Input Fields

**Formal Specification**

MergeInputFields(fields):

- If any {field} in {fields} is marked with `@inaccessible`
  - Return null
- Let {firstField} be the first field in {fields}.
- Let {fieldName} be the name of {firstField}.
- Let {fieldType} be the type of {firstField}.
- Let {description} be the description of {firstField}.
- Let {defaultValue} be the default value of {firstField} or undefined if none
  exists.
- For each {field} in {fields}:
  - Assert: {field} is **not** marked with `@inaccessible`
  - Let {type} be the type of {field}.
  - Set {fieldType} to be the result of {MostRestrictiveType(fieldType, type)}.
  - If {description} is null:
    - Let {description} be the description of {field}.
  - If {defaultValue} is undefined:
    - Set {defaultValue} to the default value of {field} or undefined if none
      exists.
- Return a new input field with the name of {fieldName}, type of {fieldType},
  and description of {description} and default value of {defaultValue}.

**Explanatory Text**

The {MergeInputFields(fields)} algorithm merges multiple input field
definitions, all sharing the same field name, into a single composed input
field. This ensures the final input type in a composed schema maintains a
consistent type, description, and default value for that field. Below is a
breakdown of how {MergeInputFields(fields)} operates:

_Inaccessible Fields_

Before calling {MergeInputFields(fields)}, all fields marked with
`@inaccessible` must be filtered out. If any such field appears in the input, it
is a precondition violation of this algorithm.

_Combining Descriptions_

The name of the merged field is taken from the first field in the list. The
description is set to the first non-`null` description encountered among the
fields. If no description is found, the merged field will have no description.

_Combining Field Types_

The merged field type is computed by calling {MostRestrictiveType(typeA,
typeB)}. Unlike output fields, where {LeastRestrictiveType(fieldTypes)} is used,
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
  minTotal: Int! = 0
}
```

In the final schema, `minTotal` is defined using the most restrictive type
(`Int!`), has a default value of `0`, and includes the description from the
original field in `Schema A`.

### Merge Argument Definitions

**Formal Specification**

MergeArgumentDefinitions(arguments):

- If any argument in {arguments} is marked with `@inaccessible`
  - Return null
- Let {mergedArgument} be the first argument in {arguments} that is not marked
  with `@require`
- If {mergedArgument} is null
  - Return null
- For each {argument} in {arguments}:
  - Assert: {argument} is **not** marked with `@inaccessible`
  - Assert: {argument} is **not** marked with `@require`
  - Set {mergedArgument} to the result of {MergeArguments(mergedArgument,
    argument)}
- Return {mergedArgument}

**Explanatory Text**

{MergeArgumentDefinitions(arguments)} merges multiple arguments that share the
same name across different field definitions into a single composed argument
definition.

_Inaccessible Arguments_

Inaccessible arguments (`@inaccessible`) should be handled and filtered out
**before** calling {MergeArgumentDefinitions(arguments)}. By the time this
algorithm is invoked, any arguments marked `@inaccessible` must already be
removed. If such an argument somehow appears here, it is a precondition
violation of this algorithm.

_Handling `@require`_

The `@require` directive is likewise handled **before** this algorithm.
Arguments marked with `@require` do not participate in the merge process and
must be filtered out of the input. If any `@require` arguments are included in
this function, it is also a precondition violation.

_Merging Arguments_

All remaining arguments (those not marked `@inaccessible` or `@require`) are
merged via {MergeArguments(mergedArgument, argument)}. This algorithm ensures
that the final composed argument is compatible with all definitions of that
argument, resolving differences in type, default value, and description.

By selectively merging differences where possible, this algorithm ensures that
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

### Merge Arguments

**Formal Specification**

MergeArguments(argumentA, argumentB):

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
- Return a new argument with the name of {argumentA}, type of {type},
  description of {description}, and default value of {defaultValue}.

**Explanatory Text**

{MergeArguments(argumentA, argumentB)} takes two arguments with the same name
but possibly differing in type, description, or default value, and returns a
single, unified argument definition.

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

Suppose we have two field definitions that share the same `limit` argument, but
differ in type, description, and default value:

```graphql example
# Schema A

type Query {
  products(limit: Int = 10): [Product]
}

# Schema B

type Query {
  products(
    """
    Number of items to fetch
    """
    limit: Int!
  ): [Product]
}

# Composed Result

type Query {
  products(
    """
    Number of items to fetch
    """
    limit: Int! = 10
  ): [Product]
}
```

### Shared Algorithms

#### Least Restrictive Type

**Formal Specification**

LeastRestrictiveType(types):

- Assert: {types} is not empty.
- Let {isNullable} be true.
- If every {type} in {types} is a non nullable type:
  - Set {isNullable} to false.
- Let {unwrappedTypes} be the list produced by replacing each non nullable type
  in {types} with its inner type.
- If any {type} in {unwrappedTypes} is a list type:
  - Assert: every {type} in {unwrappedTypes} is a list type.
  - Let {innerTypes} be the list of inner types of each type in
    {unwrappedTypes}.
  - Let {innerType} be {LeastRestrictiveType(innerTypes)}.
  - If {isNullable} is true:
    - Return {innerType} as a nullable list type.
  - Otherwise:
    - Return {innerType} as a non nullable list type.
- Otherwise:
  - Let {namedType} be {LeastRestrictiveNamedOutputType(unwrappedTypes)}.
  - If {isNullable} is true:
    - Return {namedType} as a nullable type.
  - Otherwise:
    - Return {namedType} as a non nullable type.

LeastRestrictiveNamedOutputType(namedTypes):

- Assert: every {type} in {namedTypes} is a named output type.
- Let {candidates} be the set of unique types in {namedTypes}.
- Let {supertypeCandidates} be the set of all {candidate} in {candidates} for
  which {IsOutputSupertype(candidate, type)} is true for every {type} in
  {namedTypes}.
- Assert: {supertypeCandidates} is not empty.
- Sort {supertypeCandidates} by:
  - the number of possible runtime object types in ascending order, with scalar
    and enum types having zero possible runtime object types.
  - the candidate type name in ascending lexical order.
- Return the first member of {supertypeCandidates}.

IsOutputSupertype(candidate, type):

- If {candidate} and {type} are the same named type:
  - Return {true}.
- If either {candidate} or {type} is a scalar or enum type:
  - Return {false}.
- If {candidate} is an object type:
  - Return {false}.
- If {type} is an object type:
  - Return {true} if {type} is a possible runtime object type of {candidate}.
  - Otherwise return {false}.
- Return {true} if every possible runtime object type of {type} is also a
  possible runtime object type of {candidate}.
- Otherwise return {false}.

**Explanatory Text**

{LeastRestrictiveType(types)} identifies a single type that safely handles all
possible _runtime values_ produced by the sources defining the types in {types}.
The algorithm considers all types together, so the selected type is independent
of source schema order. If one source can return `null` while another cannot,
the merged type becomes nullable to avoid runtime exceptions - because a
strictly non-null signature would be violated whenever `null` appears.
Similarly, if all sources enforce non-null, the result remains non-null.

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

_Named Output Types_

When the unwrapped types are leaf types, the algorithm requires the same scalar
or enum type. If they differ (e.g., `String` vs. `Int`), the schemas are
fundamentally incompatible for merging, yet the pre merge validation should have
already caught this issue.

When the unwrapped types are object, interface, or union types, the algorithm
selects one of the declared return types that is a supertype of every other
declared return type. A supertype candidate covers another composite type when
it can represent every possible runtime object type of that type. Other than
exact equality, object types are not supertype candidates for interface or union
types. The most specific covering candidate is selected by choosing the
candidate with the smallest possible runtime object type set, with remaining
ties broken by type name. This ensures that field type selection is
deterministic and does not depend on source schema order.

**Examples**

In the following scenario, one source might return `null`, so the resulting
merged type must allow `null`.

```graphql example
# Schema A
type Product {
  price: Float!
}

# Schema B
type Product {
  price: Float
}

# Merged Result
type Product {
  price: Float
}
```

Here, both sources use lists of `Int`, but they differ in nullability.
Consequently, the merged list type is `[Int]`, which permits a `null` list or
`null` elements.

```graphql example
# Schema A
type Product {
  ratings: [Int]!
}

# Schema B
type Product {
  ratings: [Int!]
}

# Merged Result
type Product {
  ratings: [Int]
}
```

Here, one source returns object type `Product` and the other returns union type
`FeaturedItem`. Since `FeaturedItem` contains `Product`, `FeaturedItem` is the
least restrictive return type regardless of source schema order.

```graphql example
# Schema A
type Query {
  featured: Product
}

type Product {
  id: ID
}

# Schema B
type Query {
  featured: FeaturedItem
}

union FeaturedItem = Product

type Product {
  id: ID
}

# Merged Result
type Query {
  featured: FeaturedItem
}

union FeaturedItem = Product

type Product {
  id: ID
}
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
the more restrictive settings (non-null list or non-null elements) are used.

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
input ProductFilter {
  currency: String!
}

# Schema B
input ProductFilter {
  currency: String
}

# Merged Result
input ProductFilter {
  currency: String!
}
```

In the following example, since one definition mandates non-null items
(`[Int!]`), it is _more restrictive_ and prevents `null` elements in the list.
Additionally, the other source mandates a non-null list (`[Int]!`). The merged
result, `[Int!]!`, preserves these constraints to ensure the field does not
accept or produce values that violate either source.

```graphql example
# Schema A
input ProductFilter {
  ratings: [Int!]
}

# Schema B
input ProductFilter {
  ratings: [Int]!
}

# Merged Result
input ProductFilter {
  ratings: [Int!]!
}
```

## Post Merge Validation

After the schema is composed, there are certain validations that are only
possible in the context of the fully merged schema. These validations verify
overall consistency: for example, ensuring that no type is left without
accessible fields, or that interfaces and their implementors remain compatible.
This stage confirms that the combined schema remains coherent when considered as
a whole.

### Validate Type System

#### No Queries

**Error Code**

`NO_QUERIES`

**Severity**

ERROR

**Formal Specification**

- Let {fields} be the set of all fields in the `Query` type of the merged
  schema.
- {fields} must not be empty.

**Explanatory Text**

This rule ensures that the composed schema includes at least one accessible
field on the root `Query` type.

In GraphQL, the `Query` type is essential as it defines the entry points for
read operations. If none of the composed schemas expose any query fields, the
composed schema would lack a root query, making it an invalid GraphQL schema.

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
# Schema B
type Query {
  review(id: ID!): Review
}

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

#### Reference To Inaccessible Type

**Error Code**

`REFERENCE_TO_INACCESSIBLE_TYPE`

**Formal Specification**

- Let {inputFields} be the set of all accessible fields of the input types in
  the composed schema.
- For each {inputField} in {inputFields}:
  - Let {namedType} be the named type that {inputField} references
  - {namedType} must be accessible.
- Let {outputFields} be the set of all accessible fields of the output types in
  the composed schema.
- For each {outputField} in {outputFields}:
  - Let {namedType} be the named type that {outputField} references
  - {namedType} must be accessible.
- Let {arguments} be the set of all accessible arguments of the output fields in
  the composed schema.
- For each {argument} in {arguments}:
  - Let {namedType} be the named type that {argument} references
  - {namedType} must be accessible.

**Explanatory Text**

In a composed schema, fields and arguments must only reference types that are
exposed. This requirement guarantees that public types do not reference
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

#### Reference To Internal Type

**Error Code**

`REFERENCE_TO_INTERNAL_TYPE`

**Formal Specification**

- Let {fields} be the set of all fields of the output types in the composed
  schema.
- For each {field} in {fields}:
  - Let {namedType} be the named type that {field} references
  - {namedType} must exist in the composed schema.

**Explanatory Text**

In a composed schema, fields must not reference internal types. This requirement
guarantees that public types do not reference internal structures which are
intended for internal use.

A valid case where a public field references another public type:

```graphql example
type Object1 {
  field1: String!
  field2: Object2
}

type Object2 {
  field3: String
}
```

Another valid case is where the field is internal in the source schema:

```graphql example
type Object1 {
  field1: String!
  field2: Object2 @internal
}

type Object2 @internal {
  field3: String
}
```

An invalid case is when a field references an internal type:

```graphql counter-example
type Object1 {
  field1: String!
  field2: Object2!
}

type Object2 @internal {
  field3: String
}
```

### Validate Composite Types

#### Empty Merged Object Type

**Error Code**

`EMPTY_MERGED_OBJECT_TYPE`

**Severity**

ERROR

**Formal Specification**

- Let {types} be the set of all object types in the composed schema.
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

#### Empty Merged Interface Type

**Error Code**

`EMPTY_MERGED_INTERFACE_TYPE`

**Severity**

ERROR

**Formal Specification**

- Let {types} be the set of all interface types in the composed schema.
- For each {type} in {types}:
  - Let {fields} be a set of all fields in {type}.
  - {fields} must not be empty.

**Explanatory Text**

For interface types defined across multiple source schemas, the merged interface
type is the superset of all fields defined in these source schemas. However, any
field marked with `@inaccessible` in any source schema is hidden and not
included in the merged interface type. An interface type with no fields, after
considering `@inaccessible` annotations, is considered empty and invalid.

**Examples**

In the following example, the merged object type `Product` is valid. It includes
all fields from both source schemas, with `price` being hidden due to the
`@inaccessible` directive in one of the source schemas:

```graphql
# Schema A
interface Product {
  name: String
  price: Int @inaccessible
}

# Schema B
interface Product {
  name: String
  inStock: Boolean
}
```

If the `@inaccessible` directive is applied to an interface type itself, the
entire merged interface type is excluded from the composite execution schema,
and it is not required to contain any fields.

```graphql
# Schema A
interface Product @inaccessible {
  name: String
  price: Int
}

# Schema B
interface Product {
  name: String
  inStock: Boolean
}
```

This counter-example demonstrates an invalid merged interface type. In this
case, `Product` is defined in two source schemas, but all fields are marked as
`@inaccessible` in at least one of the source schemas, resulting in an empty
merged interface type:

```graphql counter-example
# Schema A
interface Product {
  name: String
  price: Int @inaccessible
}

# Schema B
interface Product {
  name: String @inaccessible
  price: Int
}
```

#### Implemented by Inaccessible

**Error Code**

`IMPLEMENTED_BY_INACCESSIBLE`

**Severity**

ERROR

**Formal Specification**

- Let {schema} be the merged composite execution schema.
- Let {types} be the set of all object and interface types in {schema}.
- For each {type} in {types}:
  - Let {implementedInterfaces} be the set of all interfaces implemented by
    {type}.
  - For each {implementedInterface} in {implementedInterfaces}:
    - Let {interfaceFields} be the set of all fields defined on
      {implementedInterface} that are visible in the merged schema.
    - For each {interfaceField} in {interfaceFields}:
      - Let {fieldName} be the name of {interfaceField}.
      - {type} must have a field with the name {fieldName}

**Explanatory Text**

This rule ensures that inaccessible fields (`@inaccessible`) on an object or
interface type are not exposed through an interface. A composite type that
implements an interface must provide public access to each field defined by the
interface. If a field on an object type is marked as `@inaccessible` but
implements an interface field that is visible in the composed schema, this
creates a contradiction: the interface contract requires that field to be
accessible, yet the implementation hides it.

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

#### Interface Object Field Requires Implement

**Error Code**

`INTERFACE_OBJECT_FIELD_REQUIRES_IMPLEMENT`

**Severity**

ERROR

**Formal Specification**

- Let {schema} be the merged composite execution schema.
- Let {objectTypes} be the set of all object types in {schema}.
- For each {objectType} in {objectTypes}:
  - Let {ancestors} be the set of interfaces implemented by {objectType},
    directly or transitively.
  - For each {fieldName} such that {fieldName} is in
    {ProjectedFieldNames(ancestor)} for some {ancestor} in {ancestors}:
    - Let {ownDeclarations} be the declarations of {fieldName} authored directly
      on {objectType} across the source schemas, excluding declarations
      annotated with `@external` or `@internal` and declarations overridden
      through an `@override` directive in another source schema.
    - If {ownDeclarations} is empty:
      - Continue to the next {fieldName}.
    - Let {projectedDeclarations} be the result of
      {EffectiveContributingDeclarations(candidates, schema)}, where
      {candidates} contains the declarations of {fieldName} on the stand-ins of
      {ancestors}, each with its interface as its contributing interface.
    - If every declaration in {ownDeclarations} is annotated with `@implement`:
      - Continue to the next {fieldName}.
    - If no declaration in {ownDeclarations} is annotated with `@implement` and
      every declaration in the union of {ownDeclarations} and
      {projectedDeclarations} is annotated with `@shareable` or is part of a
      `@key` directive on its declaring type:
      - Continue to the next {fieldName}.
    - For each declaration in {ownDeclarations} that is not annotated with
      `@implement`:
      - Produce an `INTERFACE_OBJECT_FIELD_REQUIRES_IMPLEMENT` error.
- Let {interfaces} be the set of all interface types in {schema}.
- For each {interface} in {interfaces}:
  - Let {ancestors} be the set of interfaces implemented by {interface},
    directly or transitively.
  - For each {fieldName} in {ProjectedFieldNames(interface)}:
    - Let {ownDeclarations} be the declarations of {fieldName} on stand-ins for
      {interface}.
    - Let {lessSpecificDeclarations} be the result of
      {EffectiveContributingDeclarations(candidates, schema)}, where
      {candidates} contains the declarations of {fieldName} on the stand-ins of
      {ancestors}, each with its interface as its contributing interface.
    - If {lessSpecificDeclarations} is empty:
      - Continue.
    - If every declaration in {ownDeclarations} is annotated with `@implement`:
      - Continue to the next {fieldName}.
    - If no declaration in {ownDeclarations} is annotated with `@implement` and
      every declaration in the union of {ownDeclarations} and
      {lessSpecificDeclarations} is annotated with `@shareable` or is part of a
      `@key` directive on its stand-in:
      - Continue to the next {fieldName}.
    - For each declaration in {ownDeclarations} that is not annotated with
      `@implement`:
      - Produce an `INTERFACE_OBJECT_FIELD_REQUIRES_IMPLEMENT` error.

ProjectedFieldNames(interface):

- Let {standIns} be the set of `@interfaceObject` type declarations, across all
  source schemas, named after {interface}.
- Let {standInFieldNames} be the set of all field names declared, and not marked
  `@internal` or `@inaccessible`, on any {standIn} in {standIns}.
- Let {keyFieldNames} be the set of all field names selected by any `@key`
  directive on any {standIn} in {standIns}.
- Return the subset of {standInFieldNames} that is not in {keyFieldNames}.

**Explanatory Text**

`@interfaceObject` lets a source schema contribute field implementations for an
interface it does not itself define. Composition projects those implementations
onto the interface's implementing types. An implementing type, or a
more-specific interface stand-in, may replace a projected implementation with
its own implementation, but doing so must be explicit. `@implement`, placed on
the replacing field, declares that replacement.

An unmarked collision is also valid when every colliding declaration is
`@shareable`. That case keeps both implementations in the effective owner set,
and the distributed executor may choose any reachable one. A collision that
declares neither replacement nor sharing fails with an
`INTERFACE_OBJECT_FIELD_REQUIRES_IMPLEMENT` error.

This can happen in two ways. First, an object type may directly declare a field
whose implementation is already projected from one of its interfaces. Second, a
more-specific interface's own `@interfaceObject` stand-in may declare a field
whose implementation is already projected from a less-specific interface's
stand-in. For example, `PhysicalProduct` implements `Product`. The `Product`
stand-in declares `weight`, and the `PhysicalProduct` stand-in may also declare
`weight`.

The error message must offer all resolutions: add `@implement` to use the direct
or more-specific implementation, mark every colliding declaration with
`@shareable` to keep the implementations interchangeable, or remove the
colliding declaration to use only the projected implementation. Implementations
should aggregate this diagnostic per interface and field. They should not repeat
it for every implementing type affected. The diagnostic should name the
interface whose stand-in supplies the projected implementation and every schema
that supplies a colliding field.

Adding a stand-in field never silently changes an implementing type's behavior.
A type with no colliding declaration simply receives the projected
implementation. A type with a collision fails this rule until the field is
annotated with `@implement`, all colliding declarations are annotated with
`@shareable`, or the field is removed. Removing the stand-in field fails
composition while other source schemas still carry `@implement` markers for it;
see
[Implement Without Projected Implementation](#sec-Implement-Without-Projected-Implementation).

**Examples**

In this counter-example, `Book.reviews` is declared directly in source schema A
while `Media`'s stand-in in source schema B projects another implementation onto
`Book`. Because `Book.reviews` is not annotated with `@implement`, composition
fails with an `INTERFACE_OBJECT_FIELD_REQUIRES_IMPLEMENT` error.

```graphql counter-example
# Source Schema A
interface Media @key(fields: "id") {
  id: ID!
  title: String!
}

type Book implements Media {
  id: ID!
  title: String!
  reviews: [Review!]!
}

type Review {
  id: ID! @shareable
  rating: Int! @shareable
}

# Source Schema B
type Media @interfaceObject @key(fields: "id") {
  id: ID!
  reviews: [Review!]!
}

type Review {
  id: ID! @shareable
  rating: Int! @shareable
}
```

The first resolution keeps `Book`'s own implementation explicit:

```graphql example
type Book implements Media {
  id: ID!
  title: String!
  reviews: [Review!]! @implement
}
```

The second resolution removes `Book`'s own field entirely and uses the
implementation projected from source schema B:

```graphql example
type Book implements Media {
  id: ID!
  title: String!
}
```

The third resolution keeps both implementations eligible by marking both
declarations with `@shareable`:

```graphql example
# Source Schema A
type Book implements Media {
  id: ID!
  title: String!
  reviews: [Review!]! @shareable
}

# Source Schema B
type Media @interfaceObject @key(fields: "id") {
  id: ID!
  reviews: [Review!]! @shareable
}
```

In this case, `@implement` is not required because `Book.reviews` does not
replace the projected implementation. Both source schemas remain effective
owners.

The same rule applies one level up the interface hierarchy. Here,
`PhysicalProduct` implements `Product`, and both have `@interfaceObject`
stand-ins that declare `weight`. Composition would project both implementations
onto types that implement `PhysicalProduct`. Because `PhysicalProduct`'s
stand-in does not mark its `weight` field `@implement`, composition fails with
an `INTERFACE_OBJECT_FIELD_REQUIRES_IMPLEMENT` error, even though no object type
is involved yet.

```graphql counter-example
# Source Schema A
interface Product @key(fields: "id") {
  id: ID!
  name: String!
}

interface PhysicalProduct implements Product @key(fields: "id") {
  id: ID!
  name: String!
}

# Source Schema B
type Product @interfaceObject @key(fields: "id") {
  id: ID!
  weight: Float!
}

# Source Schema C
type PhysicalProduct @interfaceObject @key(fields: "id") {
  id: ID!
  weight: Float!
}
```

Adding `@implement` to `PhysicalProduct`'s own `weight` field resolves the
collision:

```graphql example
# Source Schema C
type PhysicalProduct @interfaceObject @key(fields: "id") {
  id: ID!
  weight: Float! @implement
}
```

Alternatively, marking both `weight` declarations with `@shareable` preserves
both projected implementations as eligible implementations. It does not move
ownership from `Product` to `PhysicalProduct`.

#### Implement Without Projected Implementation

**Error Code**

`IMPLEMENT_WITHOUT_PROJECTED_IMPLEMENTATION`

**Severity**

ERROR

**Formal Specification**

- Let {schema} be the merged composite execution schema.
- Let {declarations} be the set of all field declarations, across all source
  schemas, annotated with `@implement`.
- For each {declaration} in {declarations}:
  - Let {fieldName} be the name of {declaration}.
  - Let {type} be the type that declares {declaration}.
  - Let {hasProjectedImplementation} be false.
  - If {type} is an object type:
    - Let {interfaces} be the interfaces implemented by {type}, directly or
      transitively.
    - If {fieldName} is in {ProjectedFieldNames(interface)} for any {interface}
      in {interfaces}:
      - Set {hasProjectedImplementation} to true.
  - Otherwise, if {type} is an `@interfaceObject` stand-in for {interface}:
    - Let {ancestors} be the interfaces implemented by {interface}, directly or
      transitively.
    - If {fieldName} is in {ProjectedFieldNames(ancestor)} for any {ancestor} in
      {ancestors}:
      - Set {hasProjectedImplementation} to true.
  - A field declared directly on an interface provides no implementation and
    therefore leaves {hasProjectedImplementation} false.
  - If {hasProjectedImplementation} is false:
    - Produce an `IMPLEMENT_WITHOUT_PROJECTED_IMPLEMENTATION` error.

**Explanatory Text**

`@implement` only has meaning when there is a projected implementation to
replace. A field may be marked `@implement` while no applicable projected
implementation exists, for example because of a typo in the field name, because
the stand-in field was removed after the marker was added, or because the marker
was placed on a real interface's own field rather than on a stand-in or an
implementing type. Composition fails in every such case: the marker states an
intent that the composite schema cannot satisfy.

Note: Removing a field from a stand-in therefore fails composition while other
source schemas still carry `@implement` markers that refer to its projected
implementation. The markers in the other source schemas are removed first; the
stand-in field is removed after.

**Examples**

In this counter-example, no source schema contributes a projected implementation
of `weight`. Unlike in the earlier example, no source schema provides a
`Product` stand-in here. `@implement` on `PhysicalProduct.weight` therefore has
nothing to replace, and composition fails.

```graphql counter-example
# Source Schema A
interface Product {
  id: ID!
  name: String!
}

interface PhysicalProduct implements Product @key(fields: "id") {
  id: ID!
  name: String!
}

# Source Schema B
type PhysicalProduct @interfaceObject @key(fields: "id") {
  id: ID!
  weight: Float! @implement
}
```

Here, composition projects `weight` from `Product`'s stand-in in source schema
B. `PhysicalProduct`'s stand-in in source schema C replaces that implementation,
marked with `@implement`, so composition succeeds.

```graphql example
# Source Schema A
interface Product @key(fields: "id") {
  id: ID!
  name: String!
}

interface PhysicalProduct implements Product @key(fields: "id") {
  id: ID!
  name: String!
}

# Source Schema B
type Product @interfaceObject @key(fields: "id") {
  id: ID!
  weight: Float!
}

# Source Schema C
type PhysicalProduct @interfaceObject @key(fields: "id") {
  id: ID!
  weight: Float! @implement
}
```

#### Invalid Projected Field Sharing

**Error Code**

`INVALID_PROJECTED_FIELD_SHARING`

**Severity**

ERROR

**Formal Specification**

- Let {schema} be the merged composite execution schema.
- {ProjectedFieldNames(interface)} is defined as in
  [Interface Object Field Requires Implement](#sec-Interface-Object-Field-Requires-Implement).
- Let {typesAndInterfaces} be the set of all object and interface types in
  {schema}.
- For each {type} in {typesAndInterfaces}:
  - Let {ancestors} be the set of interfaces implemented by {type}, directly or
    transitively.
  - Let {ownFieldNames} be the set of field names {type} declares directly (an
    object type's own fields, or, if {type} is an interface, the field names
    declared on {type}'s own `@interfaceObject` stand-in).
  - Let {candidateFieldNames} be the set of field names in
    {ProjectedFieldNames(ancestor)} for some {ancestor} in {ancestors},
    excluding any name in {ownFieldNames}.
  - For each {fieldName} in {candidateFieldNames}:
    - Let {candidates} be the declarations of {fieldName} on the stand-ins of
      {ancestors}, each with its interface as its contributing interface.
    - Let {effectiveDeclarations} be the result of
      {EffectiveContributingDeclarations(candidates, schema)}.
    - Let {minimalDeclarations} be the subset of {effectiveDeclarations} for
      which no other declaration in {effectiveDeclarations} has a different
      contributing interface that implements the contributing interface of that
      declaration in {schema}.
    - If {minimalDeclarations} has fewer than 2 elements:
      - Continue.
    - Every declaration in {minimalDeclarations} must be annotated with
      `@shareable`.

**Explanatory Text**

A type may implement two or more interfaces that are not related to each other
by implementation. Stand-ins for more than one of those interfaces may declare
the same field. In that case there is no single most-specific projected
implementation to choose. More than one minimal effective declaration remains,
and composition cannot resolve the ambiguity on its own.
[Invalid Field Sharing](#sec-Invalid-Field-Sharing) addresses the same situation
for ordinary fields declared directly in multiple source schemas. This rule
applies the same principle to implementations projected from unrelated
interfaces. Composition fails with an `INVALID_PROJECTED_FIELD_SHARING` error
unless every contributing declaration is marked `@shareable`. The `@shareable`
marker signals that the contract is intentionally identical across all of them.
A declaration marked with `@implement` removes implementations projected from
less-specific interfaces, but it does not remove an implementation projected
from an unrelated interface.

Key fields are never affected by this rule. A field used in a type's `@key` is
already implicitly shareable (see the `@key` directive). A field the target
interface itself declares directly provides no implementation. The ambiguity can
only arise for a field whose implementations are contributed solely by unrelated
interfaces' stand-ins.

Implementations should report a single diagnostic per conflicting interface pair
and field. They should not report one for every type that happens to implement
both interfaces. The diagnostic should name both contributing schemas.

**Examples**

In this counter-example, `PhysicalProduct` and `DigitalProduct` are unrelated
interfaces (neither implements the other), and both stand-ins declare `reviews`.
Composition projects both implementations onto `Chair`. Both declarations are
minimal for `reviews`, and neither is `@shareable`. Composition fails with an
`INVALID_PROJECTED_FIELD_SHARING` error.

```graphql counter-example
# Source Schema A
interface PhysicalProduct @key(fields: "id") {
  id: ID!
}

interface DigitalProduct @key(fields: "id") {
  id: ID!
}

type Chair implements PhysicalProduct & DigitalProduct {
  id: ID!
}

# Source Schema B
type PhysicalProduct @interfaceObject @key(fields: "id") {
  id: ID!
  reviews: [Review!]!
}

type Review {
  id: ID! @shareable
  rating: Int! @shareable
}

# Source Schema C
type DigitalProduct @interfaceObject @key(fields: "id") {
  id: ID!
  reviews: [Review!]!
}

type Review {
  id: ID! @shareable
  rating: Int! @shareable
}
```

Marking both declarations `@shareable` tells composition that the two contracts
are intentionally identical, and `Chair.reviews` resolves without ambiguity.

```graphql example
# Source Schema B
type PhysicalProduct @interfaceObject @key(fields: "id") {
  id: ID!
  reviews: [Review!]! @shareable
}

type Review {
  id: ID! @shareable
  rating: Int! @shareable
}

# Source Schema C
type DigitalProduct @interfaceObject @key(fields: "id") {
  id: ID!
  reviews: [Review!]! @shareable
}

type Review {
  id: ID! @shareable
  rating: Int! @shareable
}
```

#### Interface Field Argument No Implementation

**Error Code**

`INTERFACE_FIELD_ARGUMENT_NO_IMPLEMENTATION`

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
    - For each {interfaceField} in {interfaceFields}:
      - If a field with the same name as {interfaceField} is not present on
        {objectType}:
        - Continue
      - Let {objectField} be the field on {objectType} with the same name as
        {interfaceField}.
      - Let {interfaceArguments} be the set of arguments on {interfaceField}.
      - For each {interfaceArgument} in {interfaceArguments}:
        - Let {argumentName} be the name of {interfaceArgument}.
        - An argument with the name {argumentName} must be present on
          {objectField}.

**Explanatory Text**

In GraphQL, an object field that implements an interface field must declare
every argument that the interface field declares. In a composite schema, this
contract can break even though every source schema is valid on its own: the
merge process removes arguments that are annotated with `@require` or
`@inaccessible` in a source schema, and an argument only survives merging if
every source schema that contributes the field declares it. If an argument is
removed from an implementing object field but survives on the merged interface
field, the composite schema would break the interface contract. This rule
detects such cases and fails the composition rather than producing an invalid
composite schema.

**Examples**

In this valid example, the interface field `Account.displayName` and the
implementing field `User.displayName` both declare the `locale` argument in the
composite schema.

```graphql example
# Schema A
interface Account {
  id: ID!
  displayName(locale: String): String
}

type User implements Account {
  id: ID!
  displayName(locale: String): String
}
```

In this counter-example, the `locale` argument on `User.displayName` is
annotated with `@require` in Schema A but not on the interface field in Schema
B, so it is removed from the implementing field but survives on the merged
interface field. The merged `User` type no longer correctly implements
`Account`, raising an `INTERFACE_FIELD_ARGUMENT_NO_IMPLEMENTATION` error.

```graphql counter-example
# Schema A
type User @key(fields: "id") {
  id: ID!
  displayName(locale: String @require(field: "preferredLocale")): String
}

# Schema B
interface Account {
  id: ID!
  displayName(locale: String): String
}

type User implements Account @key(fields: "id") {
  id: ID!
  displayName(locale: String): String
  preferredLocale: String
}
```

### Validate Input Types

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
  name: String
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
    - {coordinate} must be in the composed schema.

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

### Validate Enums

#### Empty Merged Enum Type

**Error Code**

`EMPTY_MERGED_ENUM_TYPE`

**Severity**

ERROR

**Formal Specification**

- Let {enumTypes} be the set of all enum types in the composite schema.
- For each {enumType} in {enumTypes}:
  - Let {values} be a set of all values in {enumType}.
  - {values} must not be empty.

**Explanatory Text**

Enum values have to be an exact match across all source schemas. If an enum
value only exists in one source schema, it has to be marked as `@inaccessible`.
Enum members that are marked as `@inaccessible` are not included in the merged
enum type. An enum type with no values is considered empty and invalid.

**Examples**

In the following example, the merged enum type `DeliveryStatus` is valid. It
includes all values from both source schemas, with `PENDING` being hidden due to
the `@inaccessible` directive in one of the source schemas:

```graphql
# Schema A
enum DeliveryStatus {
  PENDING @inaccessible
  SHIPPED
  DELIVERED
}

# Schema B
enum DeliveryStatus {
  SHIPPED
  DELIVERED
}
```

If the `@inaccessible` directive is applied to an enum type itself, the entire
merged enum type is excluded from the composite execution schema, and it is not
required to contain any values.

```graphql
# Schema A
enum DeliveryStatus @inaccessible {
  SHIPPED
  DELIVERED
}

# Schema B
enum DeliveryStatus {
  SHIPPED
  DELIVERED
}
```

This counter-example demonstrates an invalid merged enum type. In this case,
`DeliveryStatus` is defined in two source schemas, but all values are marked as
`@inaccessible` in at least one of the source schemas, resulting in an empty
merged enum type:

```graphql counter-example
# Schema A
enum DeliveryStatus {
  PENDING @inaccessible
  DELIVERED
}

# Schema B
enum DeliveryStatus {
  PENDING
  DELIVERED @inaccessible
}
```

#### Enum Type Default Value Inaccessible

**Error Code**

`ENUM_TYPE_DEFAULT_VALUE_INACCESSIBLE`

**Formal Specification**

- {ValidateArgumentDefaultValues()} must be true.
- {ValidateInputFieldDefaultValues()} must be true.

ValidateArgumentDefaultValues():

- Let {arguments} be the set of all arguments of fields in the composed schema
- For each {argument} in {arguments}
  - If {argument} has a default value:
    - Let {defaultValue} be the default value of {argument}
    - If not {ValidateDefaultValue(defaultValue)}
      - return false
- return true

ValidateInputFieldDefaultValues():

- Let {inputFields} be the set of all input fields in the composed schema
- For each {inputField} in {inputFields}:
  - If {inputField} has a default value:
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
  - For each {objectField} in {objectFields}:
    - Let {value} be the value of {objectField}
    - If {ValidateDefaultValue(value)} is false
      - return false
- If {defaultValue} is an EnumValue:
  - If {enum} be the enum type of {defaultValue}
  - If {enum} does not have a value with the name of {defaultValue}
    - return false
- return true

**Explanatory Text**

This rule ensures that inaccessible enum values are not exposed in the composed
schema through default values. Output field arguments and input fields must only
use enum values as their default value when not annotated with the
`@inaccessible` directive.

In this example the `FOO` value in the `Enum1` enum is not marked with
`@inaccessible`, hence it does not violate the rule.

```graphql
# Schema A
type Query {
  field(type: Enum1 = FOO): [Baz!]!
}

enum Enum1 {
  FOO
  BAR
}
```

The following example violates this rule because the default value for the
argument (`arg`) and the input field (`field`) references an enum value (`FOO`)
that is marked as `@inaccessible`.

```graphql counter-example
# Schema A
type Query {
  field(arg: Enum1 = FOO): [Baz!]!
}

input Input1 {
  field: Enum1 = FOO
}

enum Enum1 {
  FOO @inaccessible
  BAR
}
```

The following example violates this rule because the default value for the
argument (`arg`) and the input field (`field2`) references an `@inaccessible`
enum value (`FOO`) within an object value.

```graphql counter-example
# Schema A
type Query {
  field(arg: Input1 = { field1: FOO }): [Baz!]!
}

input Input1 {
  field1: Enum1
  field2: Input2 = { field3: FOO }
}

input Input2 {
  field3: Enum1
}

enum Enum1 {
  FOO @inaccessible
  BAR
}
```

The following example violates this rule because the default value for the
argument (`arg`) and the input field (`field`) references an `@inaccessible`
enum value (`FOO`) within a list.

```graphql counter-example
# Schema A
type Query {
  field(arg: [Enum1] = [FOO]): [Baz!]!
}

input Input1 {
  field: [Enum1] = [FOO]
}

enum Enum1 {
  FOO @inaccessible
  BAR
}
```

### Validate Union Types

#### Empty Merged Union Type

**Error Code**

`EMPTY_MERGED_UNION_TYPE`

**Severity**

ERROR

**Formal Specification**

- Let {unionTypes} be the set of all union types in the composite schema.
- For each {unionType} in {unionTypes}:
  - Let {members} be a set of all member types in {unionType}.
  - {members} must not be empty.

**Explanatory Text**

For union types defined across multiple source schemas, the merged union type is
the union of all member types defined in these source schemas. However, any
member type marked with `@inaccessible` in any source schema is hidden and not
included in the merged union type. A union type with no members, after
considering `@inaccessible` annotations, is considered empty and invalid.

**Examples**

In the following example, the merged union type `SearchResult` is valid. It
includes all member types from both source schemas, with `User` being hidden due
to the `@inaccessible` directive in one of the source schemas:

```graphql
# Schema A
union SearchResult = User | Product

type User @inaccessible {
  id: ID!
}

type Product {
  id: ID!
}

# Schema B
union SearchResult = Product | Order

type Product {
  id: ID!
}

type Order {
  id: ID!
}

# Composite Schema
union SearchResult = Product | Order
```

If the `@inaccessible` directive is applied to a union type itself, the entire
merged union type is excluded from the composite execution schema, and it is not
required to contain any members.

```graphql
# Schema A
union SearchResult @inaccessible = User | Product

type User {
  id: ID!
}

type Product {
  id: ID!
}

# Schema B
union SearchResult = Product | Order

type Product {
  id: ID!
}

type Order {
  id: ID!
}
```

This counter-example demonstrates an invalid merged union type. In this case,
`SearchResult` is defined in two source schemas, but all member types are marked
as `@inaccessible` in at least one of the source schemas, resulting in an empty
merged union type:

```graphql counter-example
# Schema A
union SearchResult = User | Product

type User @inaccessible {
  id: ID!
}

type Product {
  id: ID!
}

# Schema B
union SearchResult = User | Product

type User {
  id: ID!
}

type Product @inaccessible {
  id: ID!
}
```

### Validate Is Directives

#### Is Invalid Fields

**Error Code**

`IS_INVALID_FIELDS`

**Severity**

ERROR

**Formal Specification**

- Let {schemas} be all source schemas.
- Let {compositeTypes} be the set of all composite types in {schemas}.
- For each {composite} in {compositeTypes}:
  - Let {fields} be the set of fields on {composite}.
  - Let {arguments} be the set of all arguments on {fields}.
  - For each {argument} in {arguments}:
    - If {argument} is **not** annotated with `@is`:
      - Continue
    - Let {schema} be the schema that defines {argument}.
    - Let {declaringField} be the field that defines {argument}.
    - Let {declaringType} be the type that defines {declaringField}.
    - Let {otherSchemas} be the set of all {schemas} excluding {schema}.
    - Let {fieldArg} be the string value of the `field` argument of the `@is`
      directive on {argument}.
    - Let {parsedFieldArg} be the parsed selection map from {fieldArg}.
    - The parsed selection map {parsedFieldArg} must satisfy the validation
      rules defined in Appendix A, Section 6.3, using:
      - {declaringType} as the initial root type.
      - The combined schema context formed by the union of {otherSchemas} as the
        schema context except all fields marked as `@internal`
      - Validation succeeds if each required field selection path can be
        resolved across this combined schema context. Individual fields in the
        selection may exist in different schemas; it is not required that all
        fields referenced by {parsedFieldArg} reside within a single schema.

**Explanatory Text**

Even if the field selection map for `@is(field: "…")` is syntactically valid,
its contents must also be valid within the composed schema. Fields must exist on
the parent type for them to be referenced by `@is`. In addition, fields
referencing unknown fields break the valid usage of `@is`, leading to an
`IS_INVALID_FIELDS` error.

**Examples**

In the following example, the `@is` directive’s `field` argument is a valid
field selection map and satisfies the rule.

```graphql example
# Schema A
type Query {
  personById(id: ID! @is(field: "id")): Person @lookup
}

type Person {
  id: ID!
  name: String
}
```

In this counter-example, the `@is` directive references a field (`unknownField`)
that does not exist on the return type (`Person`), causing an
`IS_INVALID_FIELDS` error.

```graphql counter-example
# Schema A
type Query {
  personById(id: ID! @is(field: "unknownField")): Person @lookup
}

type Person {
  id: ID!
  name: String
}
```

Note: An `@is` selection map must not supply arguments (see
[Is Fields Has Arguments](#sec-Is-Fields-Has-Arguments)).

### Validate Require Directives

#### Require Invalid Fields

**Error Code**

`REQUIRE_INVALID_FIELDS`

**Severity**

ERROR

**Formal Specification**

- Let {schemas} be all source schemas.
- Let {compositeTypes} be the set of all composite types in {schemas}.
- For each {composite} in {compositeTypes}:
  - Let {fields} be the set of fields on {composite}.
  - Let {arguments} be the set of all arguments on {fields}.
  - For each {argument} in {arguments}:
    - If {argument} is **not** annotated with `@require`:
      - Continue
    - Let {schema} be the schema that defines {argument}.
    - Let {declaringField} be the field that defines {argument}.
    - Let {declaringType} be the type that defines {declaringField}.
    - Let {otherSchemas} be the set of all {schemas} excluding {schema}.
    - Let {fieldArg} be the string value of the `field` argument of the
      `@require` directive on {argument}.
    - Let {parsedFieldArg} be the parsed selection map from {fieldArg}.
    - The parsed selection map {parsedFieldArg} must satisfy the validation
      rules defined in Appendix A, Section 6.3, using:
      - {declaringType} as the initial root type.
      - The combined schema context formed by the union of {otherSchemas} as the
        schema context except all fields marked as `@internal`
      - Validation succeeds if each required field selection path can be
        resolved across this combined schema context. Individual fields in the
        selection may exist in different schemas; it is not required that all
        fields referenced by {parsedFieldArg} reside within a single schema.

**Explanatory Text**

Even if the selection map for `@require(field: "…")` is syntactically valid, its
contents must also be valid. Required fields must exist on the parent type in a
**different schema than the one defining the requirement** for them to be
referenced by `@require`. Additionally, requiring unknown fields invalidates
`@require`, resulting in a `REQUIRE_INVALID_FIELDS` error.

`@require` on a field declared by an `@interfaceObject` stand-in is ordinary
`@require`. The stand-in is an object type like any other, so no special rule is
needed. Its selection map is validated exactly as above, resolved against the
combined schema context of the other source schemas. Required arguments are
excluded from the composite schema, per the existing behavior of `@require`.
More than one source schema may contribute the same field name to the same
interface. This can happen through the interface's own declaration, or through
more than one of its `@interfaceObject` stand-ins. In every such case, the
argument definitions across those schemas must still be mergeable; see
[Field With Missing Required Arguments](#sec-Field-With-Missing-Required-Arguments)
for an example of this interaction.

**Examples**

In the following example, the `@require` directive's `field` argument is a valid
selection set and satisfies the rule.

```graphql example
# Schema A
type User @key(fields: "id") {
  id: ID!
  profile(name: String @require(field: "name")): Profile
}

type Profile {
  id: ID!
  name: String
}

# Schema B
type User @key(fields: "id") {
  id: ID!
  name: String
}
```

In this counter-example, the `@require` directive references a field
(`unknownField`) that does not exist on the parent type (`Book`), causing a
`REQUIRE_INVALID_FIELDS` error.

```graphql counter-example
type Book {
  id: ID!
  pages(pageSize: Int @require(field: "unknownField")): Int
}
```

In the following counter-example, the `@require` directive references a field
from itself (`Book.size`) which is not allowed. This results in a
`REQUIRE_INVALID_FIELDS` error.

```graphql counter-example
type Book {
  id: ID!
  size: Int
  pages(pageSize: Int @require(field: "size")): Int
}
```

The `@require` directive may also reference fields with arguments. In the
following example, the `weight` argument is required from the `weight` output
field selected with a constant `unit` argument:

```graphql example
# Schema A
type Product @key(fields: "id") {
  id: ID!
  shippingCost(
    weight: Float @require(field: "weight(unit: IMPERIAL)")
  ): Currency
}

# Schema B
type Product @key(fields: "id") {
  id: ID!
  weight(unit: WeightUnit!): Float
}
```

Argument values within `@require` must be constant literals; variables are not
permitted. Argument names must exist on the referenced field, values must coerce
to the argument's type, and required arguments without defaults must be
supplied. When the referenced field is defined in multiple source schemas, the
argument definitions across those schemas must be mergeable as defined by
[Field Argument Types Mergeable](#sec-Field-Argument-Types-Mergeable).

## Validate Satisfiability

The final step confirms that the composite schema supports executable queries
without leading to invalid conditions. Each query path defined in the merged
schema is checked to ensure that every field can be resolved. If any query path
is unresolvable, the schema is deemed unsatisfiable, and composition fails.

### Unsatisfiable Query Path

**Error Code**

`UNSATISFIABLE_QUERY_PATH`

**Severity**

ERROR

**Formal Specification**

- Let {schema} be the merged composite execution schema.
- Let {sourceSchemas} be the set of source schemas used to compose {schema}.
- Let {operationRootTypes} be the set containing `Query`, `Mutation`, and
  `Subscription` when defined in {schema}.
- For each {rootType} in {operationRootTypes}:
  - Let {paths} be the result of `CollectExecutablePaths(rootType, schema)`.
  - For each {path} in {paths}:
    - Let {options} be the result of `PlanOptions(path, sourceSchemas)`.
    - {options} must not be empty.
- Let {standInBindings} be the set of tuples ({interface}, {standInSchema})
  where {interface} is an interface type in {schema} and {standInSchema} is a
  source schema that defines an object type annotated with `@interfaceObject`
  whose name is the name of {interface} (the _stand-in_ for {interface}).
- For each ({interface}, {standInSchema}) in {standInBindings}:
  - Let {positions} be the result of
    `CollectOpaquePositions(interface, standInSchema)`.
  - For each {position} in {positions}:
    - If `DemandsTypeContext(position, interface, schema)` is true or
      `DemandsNonLocalData(position, interface, standInSchema, schema)` is true:
      - `HasCoveringLookup(interface, standInSchema, schema, sourceSchemas)`
        must be true.
  - `ProjectedFieldsPlannable(interface, standInSchema, schema, sourceSchemas)`
    must be true.

CollectExecutablePaths(rootType, schema):

Starting from {rootType}, collect each executable field path that can be formed
in {schema}. Each collected path is represented as an ordered list of tuples
({type}, {field}).

- Let {paths} be an empty set.
- Let {stack} be the set of one-element paths `[(rootType, fieldName)]` for each
  field named {fieldName} on {rootType} in {schema}.
- While {stack} is not empty:
  - Remove one {path} from {stack}.
  - Add {path} to {paths}.
  - Let ({parentType}, {parentField}) be the last element in {path}.
  - Let {returnType} be the unwrapped return type of {parentField} on
    {parentType} in {schema}.
  - If {returnType} is an object type:
    - Let {possibleTypes} be the set containing only {returnType}.
  - Otherwise if {returnType} is an interface or union type:
    - Let {possibleTypes} be `GetPossibleTypes(returnType)`.
  - Otherwise:
    - Continue to the next {path}.
  - For each {nextType} in {possibleTypes}:
    - Let {nextFields} be the set of field names on {nextType} in {schema}.
    - For each {nextField} in {nextFields}:
      - Let {nextElement} be the tuple ({nextType}, {nextField}).
      - If {nextElement} is already in {path}:
        - Continue to the next {nextField}.
      - Let {extendedPath} be {path} followed by {nextElement}.
      - Add {extendedPath} to {stack}.
- return {paths}.

PlanOptions(path, allSchemas):

Given {path}, determine which source schemas can satisfy it. Candidate schemas
are initialized from the first path element, then refined for the remaining
elements via `RefinePlanOptions`.

- Let {pathElements} be the ordered list of tuples ({type}, {field}) in {path}.
- If {pathElements} is empty:
  - return an empty set.
- Let ({initialType}, {initialField}) be the first element in {pathElements}.
- Let {initialOptions} be an empty set.
- Let {initialResolvers} be
  `FieldResolverOptions(initialType, initialField, allSchemas)`.
- For each ({schema}, {localType}) in {initialResolvers}:
  - Add {schema} to {initialOptions}.
- If {initialOptions} is empty:
  - return an empty set.
- Let {remainingPath} be {pathElements} without the first element.
- return `RefinePlanOptions(remainingPath, initialOptions, allSchemas)`.

FieldResolverOptions(type, field, allSchemas):

Returns the schemas and schema-local parent types that may resolve {field} on
{type}. A projected field can be declared on a stand-in whose local type name is
the name of an interface rather than {type}.

- Let {options} be an empty set of ({schema}, {localType}) tuples.
- If composition recorded an effective owner set for ({type}, {field}):
  - For each {ownerDeclaration} in that owner set:
    - Let {schema} be the source schema defining {ownerDeclaration}.
    - Let {localType} be the object type declaring {ownerDeclaration} in
      {schema}.
    - Add ({schema}, {localType}) to {options}.
  - Return {options}.
- For each {schema} in {allSchemas}:
  - If {schema} defines {field} on {type} and does not annotate it with
    `@external`:
    - Add ({schema}, {type}) to {options}.
- Return {options}.

RefinePlanOptions(pathElements, currentOptions, allSchemas):

For the remaining {pathElements}, evaluate one element at a time against
{currentOptions}. The result is the set of schemas that can satisfy the full
remainder of the path.

- If {pathElements} is empty:
  - return {currentOptions}.
- Let ({currentType}, {currentField}) be the first element in {pathElements}.
- Let {nextOptions} be an empty set.
- Let {fieldResolvers} be
  `FieldResolverOptions(currentType, currentField, allSchemas)`.
- For each {currentSchema} in {currentOptions}:
  - For each ({candidateSchema}, {localType}) in {fieldResolvers}:
    - If {candidateSchema} is not equal to {currentSchema}:
      - If `IsReachable(currentSchema, candidateSchema, localType, allSchemas)`
        is false:
        - Continue to the next {candidateSchema}.
    - If {FieldHasRequirements(candidateSchema, localType, currentField)}:
      - If
        `ResolveRequirements(currentSchema, candidateSchema, localType, currentField, allSchemas)`
        is false:
        - Continue to the next {candidateSchema}.
    - Add {candidateSchema} to {nextOptions}.
- If {nextOptions} is empty:
  - return an empty set.
- Let {remainingPath} be {pathElements} without the first element.
- return `RefinePlanOptions(remainingPath, nextOptions, allSchemas)`.

IsReachable(sourceSchema, targetSchema, type, allSchemas):

Execution can transition from {sourceSchema} to {targetSchema} for {type} only
when {targetSchema} has a compatible `@lookup` whose required input paths are
resolvable from the current context.

- Let {lookups} be the set of all fields in {targetSchema} that are annotated
  with `@lookup` and resolve {type}. A lookup resolves {type} if:
  - Its unwrapped return type is {type}, or
  - Its unwrapped return type is an interface or union whose possible object
    types include {type}.
- For each {lookup} in {lookups}:
  - Let {lookupPathSets} be `LookupPathSets(lookup, type)`.
  - For each {lookupPathSet} in {lookupPathSets}:
    - If `IsPathSetResolvable(lookupPathSet, sourceSchema, allSchemas)` is true:
      - return true.
- return false.

FieldHasRequirements(schema, type, field):

A field is requirement-bearing when at least one argument on {field} of {type}
in {schema} is annotated with `@require`.

- Let {arguments} be the set of arguments on {field} of {type} in {schema}.
- If any argument in {arguments} is annotated with `@require`:
  - return true.
- return false.

ResolveRequirements(sourceSchema, targetSchema, type, field, allSchemas):

Each `@require` argument on {field} must be resolvable from allowed schemas
(excluding {targetSchema}) when execution starts from {sourceSchema}.

- Let {allowedSchemas} be {allSchemas} excluding {targetSchema}.
- Let {requiredArguments} be the set of arguments on {field} that are annotated
  with `@require`.
- For each {requiredArgument} in {requiredArguments}:
  - Let {fieldSelectionMap} be the `field` argument value of `@require` on
    {requiredArgument}.
  - Let {requirementPathSets} be
    `ExtractPathSets(fieldSelectionMap, type, requiredArgument)`.
  - Let {argumentSatisfied} be false.
  - For each {requirementPathSet} in {requirementPathSets}:
    - If `IsPathSetResolvable(requirementPathSet, sourceSchema, allowedSchemas)`
      is true:
      - Set {argumentSatisfied} to true.
      - Break.
  - If {argumentSatisfied} is false:
    - return false.
- return true.

LookupPathSets(lookup, rootType):

For {lookup}, each argument contributes one or more `FieldSelectionMap`
alternatives. Combine those argument-level alternatives into conjunctive
path-set requirements rooted at {rootType}.

- Let {pathSets} be a set containing one empty path set.
- Let {arguments} be the set of arguments on {lookup}.
- For each {argument} in {arguments}:
  - If {argument} has an `@is` directive:
    - Let {fieldSelectionMap} be the `field` argument value of `@is`.
  - Otherwise:
    - Let {fieldSelectionMap} be the name of {argument}.
  - Let {argumentPathSets} be
    `ExtractPathSets(fieldSelectionMap, rootType, argument)`.
  - Set {pathSets} to the cartesian product of {pathSets} and
    {argumentPathSets}, where each combined element is the union of both path
    sets.
- return {pathSets}.

ExtractPathSets(fieldSelectionMap, rootType, argument):

Interpret {fieldSelectionMap} for {argument} according to Appendix A and
materialize the path-set alternatives rooted at {rootType}.

- Let {pathSets} be the set of path sets represented by {fieldSelectionMap} for
  {argument}, rooted at {rootType}, according to Appendix A:
  - Each path in a path set is represented as a list of tuples ({type}, {field})
    that starts at {rootType}.
  - A single path set represents one conjunction of required paths.
  - Multiple path sets represent alternatives (for example, from `|`).
- return {pathSets}.

IsPathSetResolvable(pathSet, sourceSchema, candidateSchemas):

A conjunctive {pathSet} is resolvable only if every path in the set has at least
one valid option when starting from {sourceSchema} and considering only
{candidateSchemas}.

- For each {path} in {pathSet}:
  - Let {options} be
    `RefinePlanOptions(path, {sourceSchema}, candidateSchemas)`.
  - If {options} is empty:
    - return false.
- return true.

CollectOpaquePositions(interface, standInSchema):

A value of {interface} is _opaque_ when it is produced through a field or lookup
of {standInSchema}. Within {standInSchema}, the name of {interface} designates
the stand-in object type. {standInSchema} therefore holds no authoritative type
identity for the value. Each opaque position is the field coordinate at which
such a value enters a query path.

- Let {standInType} be the object type in {standInSchema} whose name is the name
  of {interface}.
- Let {positions} be an empty set.
- For each {field} in all field definitions in {standInSchema}, including fields
  annotated with `@lookup`:
  - Let {returnType} be the unwrapped return type of {field}.
  - If {returnType} is {standInType}:
    - Let {parentType} be the type declaring {field}.
    - Add the tuple ({parentType}, {field}) to {positions}.
- return {positions}.

DemandsTypeContext(position, interface, schema):

Type context is demanded at {position} when a client-executable selection on the
opaque value can observe its concrete type, either by selecting `__typename` or
by applying a type condition that narrows {interface} to one of its possible
types. Because every client-executable selection set on an interface-typed value
may select `__typename`, type context is demanded at every opaque position that
clients can reach.

- Let ({parentType}, {field}) be {position}.
- If {field} is not accessible in the client-facing composite schema:
  - return false.
- return true.

DemandsNonLocalData(position, interface, standInSchema, schema):

Non-local data is demanded at {position} when a client-executable selection on
the opaque value can include a field that {standInSchema} does not itself
declare on the stand-in type. Such a field must be resolved by another source
schema. Resolving it first requires recovering the value's identity.

- Let ({parentType}, {field}) be {position}.
- If {field} is not accessible in the client-facing composite schema:
  - return false.
- Let {standInType} be the object type in {standInSchema} whose name is the name
  of {interface}.
- Let {localFieldNames} be the set of field names on {standInType} in
  {standInSchema}.
- For each {fieldDefinition} in the fields of {interface} in {schema}:
  - If the name of {fieldDefinition} is not in {localFieldNames}:
    - return true.
- For each {possibleType} in `GetPossibleTypes(interface)` in {schema}:
  - For each {fieldDefinition} in the fields of {possibleType} in {schema}:
    - If the name of {fieldDefinition} is not in {localFieldNames}:
      - return true.
- return false.

HasCoveringLookup(interface, standInSchema, schema, sourceSchemas):

Identity and non-local data for an opaque value can only be recovered through a
_covering interface lookup_: a lookup, in a single source schema, that returns
{interface} itself. This lookup must be reachable with the stand-in's key
fields. Its schema-local possible-type set for {interface} must contain every
possible type of {interface} in the composite schema.

- Let {compositePossibleTypes} be `GetPossibleTypes(interface)` in {schema}.
- For each {candidateSchema} in {sourceSchemas}:
  - If {candidateSchema} does not define {interface} as an interface type:
    - Continue to the next {candidateSchema}.
  - Let {localPossibleTypes} be `GetPossibleTypes(interface)` in
    {candidateSchema}.
  - If {localPossibleTypes} is not a superset of {compositePossibleTypes}:
    - Continue to the next {candidateSchema}.
  - Let {lookups} be the set of fields in {candidateSchema} annotated with
    `@lookup` whose unwrapped return type is {interface}.
  - For each {lookup} in {lookups}:
    - Let {lookupPathSets} be `LookupPathSets(lookup, interface)`.
    - For each {lookupPathSet} in {lookupPathSets}:
      - If `IsPathSetResolvable(lookupPathSet, standInSchema, sourceSchemas)` is
        true:
        - return true.
- return false.

ProjectedFieldsPlannable(interface, standInSchema, schema, sourceSchemas):

Each non-key field of the stand-in is projected onto every implementing type as
an implementation of that field. When the stand-in remains its only effective
owner, the stand-in schema must be reachable through one of its own lookups.
When the field is shareable, at least one effective owner must be reachable.
This must hold from every context that can resolve values of that type.

- Let {standInType} be the object type in {standInSchema} whose name is the name
  of {interface}.
- Let {keyFieldNames} be the set of field names selected by the `@key`
  directives on {standInType}.
- Let {projectedFields} be the set of fields on {standInType} whose name is not
  in {keyFieldNames}.
- For each {field} in {projectedFields}:
  - For each {possibleType} in `GetPossibleTypes(interface)` in {schema}:
    - Let {ownerDeclarations} be the effective owner set recorded for
      ({possibleType}, {field}).
    - If the declaration of {field} on {standInType} in {standInSchema} is not
      in {ownerDeclarations}:
      - Continue to the next {possibleType}.
    - For each {sourceSchema} in {sourceSchemas}:
      - If {sourceSchema} does not define {possibleType} as an object type:
        - Continue to the next {sourceSchema}.
      - Let {hasReachableOwner} be false.
      - For each {ownerDeclaration} in {ownerDeclarations}:
        - Let {ownerSchema} be the schema defining {ownerDeclaration}.
        - Let {ownerType} be the object type declaring {ownerDeclaration}. This
          is either {possibleType} for a direct declaration or a stand-in type
          for a projected declaration.
        - If {sourceSchema} is {ownerSchema} or
          `IsReachable(sourceSchema, ownerSchema, ownerType, sourceSchemas)` is
          true:
          - Set {hasReachableOwner} to true.
          - Break.
      - If {hasReachableOwner} is false:
        - return false.
- return true.

**Explanatory Text**

The satisfiability phase must ensure that every executable field path in the
composed API can be fulfilled by at least one valid query plan.
`PlanOptions(path, allSchemas)` computes this by walking the path step-by-step
and tracking which source schemas can resolve each step.

For an ordinary field, resolver options are the source schemas that declare the
field on the path's current type. For a field projected from an
`@interfaceObject`, resolver options come from the effective owner set recorded
during merge. A projected owner declares the field on its stand-in type, while a
direct owner declares it on the implementing object type. A shareable field
therefore contributes one planning option for every effective owner, and the
planner retains whichever options are reachable in the current context.

When the algorithm stays within the same source schema, it continues directly.
When it switches to another schema, the transition is valid only if the target
schema is reachable via a compatible `@lookup` and all required lookup inputs
can themselves be resolved from the current schema context.

Likewise, if a field declares `@require` dependencies, those dependencies must
also be resolvable from schemas other than the one defining that requirement.
`FieldSelectionMap` alternatives are handled as path-set alternatives where at
least one alternative must resolve. If no alternative resolves, that candidate
schema is removed from the options for that path step.

If every candidate is eliminated for any field path, the path is unsatisfiable
and composition fails with `UNSATISFIABLE_QUERY_PATH`.

A source schema defines a field marked with `@external` but does not resolve it;
external fields are therefore never resolution candidates in the source schema
that declares them.

The `@provides` directive is an execution-time optimization that allows a source
schema to return external fields as part of the same response when resolving the
annotated field. Each `@provides` selection must itself be deliverable by the
providing source schema, which is enforced by the `@provides` validation rules.
Query-path satisfiability, however, is evaluated as if all `@provides`
directives were ignored: a `@provides` may reduce the number of fetches in a
query plan, but must never be required to make a query path satisfiable.

_Opaque Values_

A source schema may declare an object type annotated with `@interfaceObject` as
the stand-in for an interface. Values produced through that schema's fields and
lookups are then opaque. Within the stand-in schema, the value is only an
instance of the local object type. That schema carries no authoritative concrete
type for the value in the composite schema. A `__typename` resolved there
returns the local object type, which is wrong in the composite schema. Values
obtained through a schema that defines the real interface are not opaque. The
demands described here arise only where a stand-in schema produces the value.
The stand-in declares its key fields as its own fields. An opaque value
therefore always carries the key fields needed to re-enter other source schemas.

_Demanding Type Context_

A query path demands type context at an opaque position when it selects
`__typename`, or when it applies a type condition that narrows the interface to
one of its possible types. Any client-executable selection on an interface-typed
value may select `__typename`. Every opaque position that is accessible in the
client-facing composite schema therefore demands type context. Positions that
exist only for the _distributed executor_, for example a lookup annotated with
`@internal`, demand none. The executor enters a stand-in schema only when the
value's identity is already known. It never requests type identity from a
stand-in schema.

_Demanding Non-Local Data_

A query path demands non-local data at an opaque position when it selects fields
beyond those the stand-in schema itself declares: interface fields contributed
by other source schemas, or fields declared by an implementing type. Such
selections must be resolved by other source schemas. Resolving them for a
specific value first requires recovering that value's identity. A selection may
instead require no type context and select only fields the stand-in declares. In
that case, a single request to the stand-in schema is a correct and complete
plan, and no recovery is needed.

_Covering Interface Lookups_

Both demands are met by the same capability. A _covering interface lookup_ for
an interface at an opaque position is a lookup, in a single source schema, that
returns the interface itself, whose required inputs are resolvable from the
stand-in's key fields, and whose schema-local possible-type set for the
interface is a superset of the composite schema's possible-type set at that
position.

The superset condition is essential because a schema's lookup can only ever
return types that the schema defines. A lookup may receive a key that identifies
a concrete type its schema does not define. In that case it cannot produce the
value. It either misreports or silently drops data. The comparison must be made
against the composite schema's possible-type set, not against the view of any
single source schema. This is because the merged implements relation is the
union of all per-schema implements edges, completed by transitive closure. Any
source schema may add an implementing type, and a derived implements edge may
widen the composite possible-type set beyond every individual schema's local
view.

Coverage must come from a single schema's lookup. Two lookups in different
schemas may have possible-type sets that are jointly, but not individually, a
superset. Such lookups do not combine. Choosing which of them to call for a
given opaque value would itself require the type identity that is being
recovered. A covering interface lookup may be annotated with `@internal`. It is
a capability of the executor, not of clients. The interface is not required to
declare a key, and no particular schema is required to provide a lookup. This
rule demands a covering interface lookup only where an opaque position gives
rise to one of the two demands above.

_Reaching Projected Fields_

Every non-key field of a stand-in supplies an implementation that composition
may project onto each implementing type. The effective owner set is resolved
during merge (see _Resolving Effective Owners_ in
[Project Interface Object Fields](#sec-Project-Interface-Object-Fields)). When
the stand-in is the only owner, its data lives only in the stand-in schema, so
one of that schema's lookups must be reachable, with the value's key fields,
from every context that can resolve values of the implementing type.

When `@shareable` preserves both a projected declaration and one or more direct
declarations, the planner may choose any reachable effective owner. The stand-in
need not be reachable from a particular context when another effective owner is
reachable from that context. Conversely, merely marking declarations as
`@shareable` does not make an otherwise unreachable owner usable.

Implementing types that use `@implement` to replace the projected implementation
remove the stand-in from the owner set and are exempt from the stand-in
reachability requirement. This includes a declaration acquired via `@override`.
Their field is resolved by one of the remaining direct owners, and the stand-in
is not consulted.

A stand-in with no lookups is always valid as a reference. It may declare its
key fields and be used to reference the entity from its own schema. If it
contributes a non-key field as the only effective owner, no plan can fetch that
field for values resolved elsewhere, so composition fails. A non-key field may
still be valid without a lookup when it is `@shareable` and another effective
owner is reachable from every context that needs the field.

_Diagnostics_

Failures raised by these clauses are reported as `UNSATISFIABLE_QUERY_PATH`
errors. The error message must name the failing query path and the missing
capability. For a missing covering interface lookup, the message must name the
interface, the opaque position, and the possible types that no single schema's
lookup covers. The composite possible-type set may have been widened by an
implements edge added during transitive-closure completion. In that case, the
message must also state the derived edge and name the source schema whose
declarations introduced it. For a projected field that no plan can reach, the
message must name the contributing schema, the interface, and the field. For
example: "Source schema B contributes fields to `Media` but provides no lookup
to resolve them."

**Examples**

The following query path:

`Query.me.profile.age`

is represented as:

`[(Query, me), (User, profile), (Profile, age)]`

Similarly, this path:

`Mutation.createUser.query.me`

is represented as:

`[(Mutation, createUser), (CreateUserPayload, query), (Query, me)]`

In the following example, source schema A defines the interface `Media` with its
implementing types, and source schema B contributes a `reviews` field to `Media`
through a stand-in:

```graphql example
# Source Schema A
interface Media {
  id: ID!
  title: String!
}

type Book implements Media {
  id: ID!
  title: String!
  author: String!
}

type Movie implements Media {
  id: ID!
  title: String!
  director: String!
}

type Query {
  mediaById(id: ID!): Media @lookup
}

# Source Schema B
type Media @interfaceObject @key(fields: "id") {
  id: ID!
  reviews: [Review!]!
}

type Review {
  body: String!
}

type Query {
  mediaById(id: ID!): Media @lookup @internal
  topReviewed(limit: Int = 10): [Media!]!
}
```

Values produced by `topReviewed` and by source schema B's `mediaById` are
opaque. The query path `[(Query, topReviewed), (Book, author)]` demands both
type context and non-local data. It demands type context because the path
narrows `Media` to `Book`. It demands non-local data because `author` is not
declared by the stand-in. Source schema A's `mediaById` is a covering interface
lookup. It returns `Media` itself. Its `id` argument is resolvable from the
stand-in's key field `id`. Its schema-local possible-type set \{`Book`,
`Movie`\} is a superset of the composite possible-type set \{`Book`, `Movie`\}.
The projected field `reviews` can be reached. Neither `Book` nor `Movie`
replaces it. Source schema B's `mediaById` is reachable with `id` from every
context that resolves `Book` or `Movie`. The composition is satisfiable.

In the following counter-example, the reviews schema contributes the non-key
field `reviews` to `Media` but provides no lookup:

```graphql counter-example
# Source Schema A
interface Media {
  id: ID!
  title: String!
}

type Book implements Media {
  id: ID!
  title: String!
}

type Movie implements Media {
  id: ID!
  title: String!
}

type Query {
  mediaById(id: ID!): Media @lookup
}

# Source Schema B
type Media @interfaceObject @key(fields: "id") {
  id: ID!
  # The stand-in contributes "reviews", but source schema B declares no
  # lookup through which the executor could ever fetch it.
  reviews: [Review!]!
}

type Review {
  body: String!
}
```

The field `reviews` is projected onto `Book` and `Movie`, so the query path
`[(Query, mediaById), (Book, reviews)]` is executable in the composite schema.
Only source schema B holds the data for `reviews`. Source schema B declares no
lookup for its stand-in, so no plan can reach it from source schema A.
Composition fails with an error such as "Source schema B contributes fields to
`Media` but provides no lookup to resolve them." Had the stand-in declared only
its key field `id`, it would have contributed no projected fields. It would have
remained a valid, reference-only stand-in.

In the following counter-example, a third schema adds an implementing type that
source schema A does not define, breaking coverage:

```graphql counter-example
# Source Schema A
interface Media {
  id: ID!
  title: String!
}

type Book implements Media {
  id: ID!
  title: String!
  author: String!
}

type Movie implements Media {
  id: ID!
  title: String!
  director: String!
}

type Query {
  mediaById(id: ID!): Media @lookup
}

# Source Schema B
type Media @interfaceObject @key(fields: "id") {
  id: ID!
  reviews: [Review!]!
}

type Review {
  body: String!
}

type Query {
  mediaById(id: ID!): Media @lookup @internal
  topReviewed(limit: Int = 10): [Media!]!
}

# Source Schema C
interface Media {
  id: ID!
  title: String!
}

# "Photo" is not defined by source schema A, so source schema A's "mediaById"
# can no longer answer for every possible type of "Media".
type Photo implements Media {
  id: ID!
  title: String!
  width: Int!
}

type Query {
  photoById(id: ID!): Photo @lookup
}
```

The composite possible-type set of `Media` is now \{`Book`, `Movie`, `Photo`\}.
Source schema A's `mediaById` has the schema-local possible-type set \{`Book`,
`Movie`\}. Source schema C defines `Media` with the schema-local possible-type
set \{`Photo`\}. No single source schema's lookup covers the composite set.
Every query path that demands type context or non-local data at an opaque
position is unsatisfiable. This includes, for example, `Query.topReviewed` with
`__typename` selected, or `[(Query, topReviewed), (Book, author)]`. The error
must name the failing path, the interface `Media`, and the uncovered possible
type `Photo`, together with source schema C, which introduced it. For example:
"The query path `Query.topReviewed` cannot be satisfied: values of `Media`
produced by source schema B are opaque, and no source schema provides a lookup
for `Media` that covers the possible type `Photo` introduced by source schema
C." Composition succeeds again once some source schema both defines every
possible type of `Media` and provides an interface lookup, for example when
source schema A also declares `type Photo implements Media` with at least its
key fields.
