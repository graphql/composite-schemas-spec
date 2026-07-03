# Source Schema

A _source schema_ is a GraphQL schema that is part of a larger _composite
schema_. Source schemas use directives to express intent and requirements for
the composition process as well as to describe runtime behavior. The following
chapters describe the directives that are used to annotate a source schema.

## @lookup

```graphql
directive @lookup on FIELD_DEFINITION
```

The `@lookup` directive is used within a _source schema_ to specify output
fields that can be used by the _distributed GraphQL executor_ to resolve an
entity by a stable key.

The stable key is defined by the arguments of the field. Each lookup argument
must match a field on the return type of the lookup field. The matched field
does not need to be defined in the source schema that declares the lookup field;
it must exist on the return type in at least one source schema. The _distributed
GraphQL executor_ resolves the key value from the source schemas where the field
is available.

Source schemas can provide multiple lookup fields for the same entity to resolve
the entity by different keys.

In this example, the source schema specifies that the `Product` entity can be
resolved with the `productById` field or the `productByName` field. Both lookup
fields are able to resolve the `Product` entity but do so with different keys.

```graphql example
type Query {
  version: Int # NOT a lookup field.
  productById(id: ID!): Product @lookup
  productByName(name: String!): Product @lookup
}

type Product {
  id: ID!
  name: String!
}
```

Lookup fields may return object, interface, or union types. In case a lookup
field returns an abstract type (interface type or union type), all possible
object types of the abstract return type are considered entities, and each must
have fields that correspond to every argument of the lookup field. When an
argument is annotated with the `@is` directive, its selection map defines this
correspondence instead; the selection map must cover every possible object type
of the return type (see [@is](#sec--is)).

```graphql example
type Query {
  product(id: ID!, categoryId: Int): Product @lookup
}

union Product = Electronics | Clothing

type Electronics {
  id: ID!
  categoryId: Int
  name: String
  brand: String
  price: Float
}

type Clothing {
  id: ID!
  categoryId: Int
  name: String
  size: String
  price: Float
}
```

The following example shows an invalid lookup field because the `Clothing` type,
which is one of the possible object types of the abstract return type, does not
define all the fields required by the lookup field’s arguments.

```graphql counter-example
type Query {
  product(id: ID!, categoryId: Int): Product @lookup
}

union Product = Electronics | Clothing

type Electronics {
  id: ID!
  categoryId: Int
  name: String
  brand: String
  price: Float
}

# Clothing does not have a field that corresponds
# with the lookup field's argument signature.
type Clothing {
  id: ID!
  name: String
  size: String
  price: Float
}
```

Lookup fields must be accessible from the `Query` type. If a lookup field is not
defined directly on the `Query` type, it must be reachable by following a chain
of fields — starting from the `Query` root type — where none of the intermediate
fields have arguments. This ensures that lookup fields are accessible to the
executor.

```graphql example
type Query {
  lookups: Lookups!
}

type Lookups {
  productById(id: ID!): Product @lookup
}

type Product {
  id: ID!
}
```

## @internal

```graphql
directive @internal on OBJECT | FIELD_DEFINITION
```

The `@internal` directive is used in combination with lookup fields and allows
you to declare internal types and fields. Internal types and fields do not
appear in the final client-facing composite schema and do not participate in the
standard schema-merging process. This allows a source schema to define lookup
fields for resolving entities that should not be accessible through the
client-facing composite schema.

```graphql example
# Source Schema
type Query {
  productById(id: ID!): Product
  productBySku(sku: ID!): Product @internal
}

# Composite Schema
type Query {
  productById(id: ID!): Product
}
```

Since internal types and fields do not participate in the standard
schema-merging process they do not collide with similar named fields or types on
other source schemas.

```graphql example
# Source Schema A
type Query {
  # this field follows the standard field merging rules
  productById(id: ID!): Product

  # this field is internal and does not follow any field merging rules.
  productBySku(sku: ID!): Product @internal
}

# Source Schema B
type Query {
  productById(id: ID!): Product
  productBySku(sku: ID!, name: String!): Product @internal
}

# Composite Schema
type Query {
  productById(id: ID!): Product
}
```

Internal fields can only be used by the distributed GraphQL executor as lookup
fields for entity resolution.

```graphql example
# Source Schema A
type Query {
  productById(id: ID!): Product @lookup
  lookups: InternalLookups! @internal
}

# all lookups within this internal type are hidden from the public API
# but can be used for entity resolution.
type InternalLookups @internal {
  productBySku(sku: ID!): Product @lookup
}

# Composite Schema
type Query {
  productById(id: ID!): Product
}
```

Since internal fields are not part of the standard schema-merging process, they
cannot be used as key fields or in requirements. This is because there is no
semantic equivalence of the field or type to another source schema.

```graphql counter-example
type Query {
  productById(id: ID!): Product @lookup
}

type Product {
  id: ID! @internal
}
```

In contrast to `@inaccessible`, the effect of `@internal` is local to its source
schema.

```graphql example
# Source Schema A
type Query {
  # this field follows the standard field merging rules
  productById(id: ID!): Product

  # this field is internal and does not follow any field merging rules.
  productBySku(sku: ID!): Product @internal
}

# Source Schema B
type Query {
  # this field follows the standard field merging rules
  productById(id: ID!): Product

  # this field follows the standard field merging rules
  productBySku(sku: Int!): Product
}

# Composite Schema
type Product {
  productById(id: ID!): Product
  productBySku(sku: Int!): Product
}
```

## @inaccessible

```graphql
# prettier-ignore
directive @inaccessible on
  | FIELD_DEFINITION
  | OBJECT
  | INTERFACE
  | UNION
  | ARGUMENT_DEFINITION
  | SCALAR
  | ENUM
  | ENUM_VALUE
  | INPUT_OBJECT
  | INPUT_FIELD_DEFINITION
```

The `@inaccessible` directive is used to prevent specific type system members
from being accessible through the client-facing _composite schema_, even if they
are accessible in the underlying source schemas.

This directive is useful for restricting access to type system members that are
either irrelevant to the client-facing composite schema or sensitive in nature,
such as internal identifiers or fields intended only for backend use.

In the following example, the key field `sku` is inaccessible from the composite
schema. However, type system members marked as `@inaccessible` can still be used
by the distributed executor to fulfill requirements.

```graphql example
type Product @key(fields: "id") @key(fields: "sku") {
  id: ID!
  sku: String! @inaccessible
  note: String
}

type Query {
  productById(id: ID!): Product
  productBySku(sku: String!): Product @inaccessible
}
```

In contrast to the `@internal` directive, `@inaccessible` hides type system
members from the composite schema even if other source schemas on the same type
system member have no `@inaccessible` directive.

```graphql example
# Source Schema A
type Product @key(fields: "id") @key(fields: "sku") {
  id: ID!
  sku: String! @inaccessible
  note: String
}

# Source Schema B
type Product @key(fields: "sku") {
  sku: String!
  price: Float!
}

# Composite Schema
type Product {
  id: ID!
  note: String
  price: Float!
}
```

## @is

```graphql
directive @is(field: FieldSelectionMap!) on ARGUMENT_DEFINITION
```

The `@is` directive is utilized on lookup fields to describe how the arguments
can be mapped from the entity type that the lookup field resolves. The mapping
establishes semantic equivalence between disparate type system members across
source schemas and is used in cases where an argument does not directly align
with a field on the entity type.

The fields referenced by an `@is` selection map must not declare arguments. The
arguments of a lookup field represent a stable key of the entity, and a stable
key must map to plain field values; a parameterized field cannot serve as part
of a lookup key.

In the following example, the directive specifies that the `id` argument on the
field `Query.personById` and the field `Person.id` on the return type of the
field are semantically the same.

Note: In cases where the lookup argument name aligns with the field name on the
return type, the `@is` directive can be omitted.

```graphql example
type Query {
  personById(productId: ID! @is(field: "id")): Person @lookup
}
```

The `@is` directive also allows referring to nested fields relative to `Person`.

```graphql example
type Query {
  personByAddressId(id: ID! @is(field: "address.id")): Person
}
```

The `@is` directive can be applied to multiple arguments within the same lookup
field, allowing each argument to be mapped individually to fields on the return
type.

```graphql example
type Query {
  personByAddressId(
    id: ID! @is(field: "address.id")
    kind: PersonKind @is(field: "kind")
  ): Person
}
```

The `@is` directive can also be used in combination with `@oneOf` to specify a
single lookup field that can resolve entities by multiple keys.

```graphql example
type Query {
  person(
    by: PersonByInput
      @is(field: "{ id } | { addressId: address.id } | { name }")
  ): Person
}

input PersonByInput @oneOf {
  id: ID
  addressId: ID
  name: String
}
```

The alternatives of an `@is` selection map represent alternative stable keys -
entry requirements for resolving an entity in this source schema. Any one of the
keys is sufficient to enter the source schema. In contrast to `@require`, where
the executor fetches the data for all alternatives and the runtime data decides
which value is used, the query planner selects which alternative it uses based
on the data it can resolve in the current context. A lookup field is usable as
long as at least one alternative can be resolved (see
[Validate Satisfiability](#sec-Validate-Satisfiability)).

When a lookup field returns an abstract type, the selection map must cover every
possible runtime type of the return type: each possible object type must be
matched by at least one alternative of the selection map. When a lookup field
declares multiple arguments, each argument must independently be mappable for
every possible runtime type. A source schema that can only resolve a subset of
the possible types must declare a narrower return type that reflects what it can
resolve.

In the following example, the selection map covers all three possible types of
`Media`, resolving each by a different key field.

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

In the following counter-example, the selection map covers only `Book` and
`Movie`. `Podcast` is a possible type of `Media` but is not covered by any
alternative, so the lookup field is invalid.

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

**Arguments:**

- `field`: Represents a selection path map syntax.

## @require

```graphql
directive @require(field: FieldSelectionMap!) on ARGUMENT_DEFINITION
```

The `@require` directive is used to express data requirements with other source
schemas. Arguments annotated with the `@require` directive are removed from the
_composite schema_ and the value for these will be resolved by the _distributed
executor_.

```graphql example
type Product {
  id: ID!
  delivery(
    zip: String!
    size: Int! @require(field: "dimension.size")
    weight: Int! @require(field: "dimension.weight")
  ): DeliveryEstimates
}
```

The above example would translate to the following in the _composite schema_.

```graphql example
type Product {
  id: ID!
  delivery(zip: String!): DeliveryEstimates
}
```

This can also be done by using input types. The selection path map specifies
which data is required and needs to be resolved from other source schemas. If
the input type is only used to express requirements it is removed from the
composite schema.

```graphql example
type Product {
  id: ID!
  delivery(
    zip: String!
    dimension: ProductDimensionInput!
      @require(field: "{ size: dimension.size, weight: dimension.weight }")
  ): DeliveryEstimates
}
```

If the input types do not match the output type structure the selection map
syntax can be used to specify how requirements translate to the input object.

```graphql example
type Product {
  id: ID!
  delivery(
    zip: String!
    dimension: ProductDimensionInput!
      @require(
        field: "{ productSize: dimension.size, productWeight: dimension.weight }"
      )
  ): DeliveryEstimates
}

type ProductDimension {
  size: Int!
  weight: Int!
}

input ProductDimensionInput {
  productSize: Int!
  productWeight: Int!
}
```

Fields referenced by a `@require` selection map may declare arguments. Unlike
`@key`, `@provides`, and `@is`, which must reference plain fields, a `@require`
selection map derives an input value and may therefore select fields with
constant arguments. Argument values must be constant literals; variables are not
permitted.

In the following example, the `weight` argument of the `shippingCost` field is
derived from the `weight` field defined in another source schema, selected with
the constant `IMPERIAL` value for the `unit` argument.

```graphql example
# Source Schema A
type Product @key(fields: "id") {
  id: ID!
  shippingCost(
    weight: Float @require(field: "weight(unit: IMPERIAL)")
  ): Currency
}

# Source Schema B
type Product @key(fields: "id") {
  id: ID!
  weight(unit: WeightUnit!): Float
}
```

The `@require` directive can also be applied to arguments of fields declared on
interface types. In this case, the selection map is rooted at the interface type
and is evaluated against the concrete runtime object. Since implementing fields
redeclare the arguments of an interface field, the `@require` annotation must be
applied consistently: an argument carries `@require` on the interface field and
on the corresponding argument of every implementing field, or on neither.
Composition then removes the argument everywhere at once, and the composite
schema retains a valid interface contract.

```graphql example
interface Product {
  id: ID!
  delivery(
    zip: String!
    size: Int! @require(field: "dimension.size")
  ): DeliveryEstimates
}

type Book implements Product {
  id: ID!
  delivery(
    zip: String!
    size: Int! @require(field: "dimension.size")
  ): DeliveryEstimates
}
```

The above example translates to the following in the composite schema; the
interface contract remains intact:

```graphql example
interface Product {
  id: ID!
  delivery(zip: String!): DeliveryEstimates
}

type Book implements Product {
  id: ID!
  delivery(zip: String!): DeliveryEstimates
}
```

Fields of specific implementing types can be referenced in the selection map of
an interface field argument through type conditions. In the following example,
the `code` argument on the interface field `Media.similar` is derived from a
field of the concrete runtime type: for a `Book` the executor supplies the
`isbn` value, and for a `Movie` the `upc` value. Each implementing field
declares the same requirement with a selection map rooted at its own type.

```graphql example
interface Media {
  id: ID!
  similar(code: String @require(field: "<Book>.isbn | <Movie>.upc")): [Media]
}

type Book implements Media {
  id: ID!
  similar(code: String @require(field: "isbn")): [Media]
}

type Movie implements Media {
  id: ID!
  similar(code: String @require(field: "upc")): [Media]
}
```

The `@require` directive must not be used on arguments of fields annotated with
`@lookup`. The arguments of a lookup field represent the stable key with which
the _distributed executor_ resolves an entity; they are supplied from an
existing representation of the entity, and a requirement has no defined meaning
in that position.

**Runtime Behavior**

At runtime, the _distributed executor_ first fetches the data for all
alternatives of the selection map and only then evaluates the map against the
fetched data to derive the argument value (see
[Value Production](#sec-Value-Production)). Which alternative supplies the value
is decided by the runtime data, not by the query planner; the query plan
requests the data for every alternative. The selection map may produce no value
for a given runtime object, for example, because a type condition does not match
the object's runtime type or because a field on an intermediate segment of the
selected path resolves to null.

If the selection map produces no value, the annotated argument is treated as if
it had not been provided, and the standard GraphQL argument coercion rules
apply: if the argument defines a default value, the default value is used; if
the argument is nullable, it remains unprovided. If the argument is non-null and
does not define a default value, the requirement cannot be satisfied and the
field cannot be invoked; a field error is raised for the annotated field and is
handled according to the standard GraphQL error propagation rules.

Note: Type conditions in a selection map are not required to cover every
possible runtime type of an abstract type. However, for a non-null argument
without a default value, an uncovered runtime type means that the annotated
field always results in a field error for objects of that type. Schema authors
should cover all possible runtime types, make the argument nullable, or provide
a default value.

**Arguments:**

- `field`: Represents a selection path map syntax.

## @key

```graphql
directive @key(fields: FieldSelectionSet!) repeatable on OBJECT | INTERFACE
```

The `@key` directive is used to designate an entity's unique key, which
identifies how to uniquely reference an instance of an entity across different
source schemas.

```graphql example
type Product @key(fields: "id") {
  id: ID!
  sku: String!
  name: String!
  price: Float!
}
```

Each occurrence of the `@key` directive on an object or interface type specifies
one distinct unique key for that entity. Keys allow the distributed GraphQL
executor to distinguish between different entities of the same type.

```graphql example
type Product @key(fields: "id") @key(fields: "sku") {
  id: ID!
  sku: String!
  name: String!
  price: Float!
}
```

While multiple keys define separate ways to reference the same entity based on
different sets of fields, a composite key allows for uniquely identifying an
entity by using a combination of multiple fields.

```graphql example
type Product @key(fields: "id sku") {
  id: ID!
  sku: String!
  name: String!
  price: Float!
}
```

The directive is applicable to both OBJECT and INTERFACE types. This allows
entities that implement an interface to inherit the key(s) defined at the
interface level, ensuring consistent identification across different
implementations of that interface.

By applying the `@key` directive all referenced fields become sharable even if
the fields are not explicitly marked with `@shareable`.

```graphql example
# source schema A
type Product @key(fields: "id") {
  id: ID!
  price: Float!
}

# source schema B
type Product @key(fields: "id") {
  id: ID!
  name: String!
}
```

Fields must be explicitly marked as a key or annotated with the `@shareable`
directive to allow multiple source schemas to define them, ensuring that the
decision to serve a field from more than one source schema is intentional and
coordinated.

```graphql counter-example
# source schema A
type Product @key(fields: "id") {
  id: ID!
  price: Float!
}

# source schema B
type Product {
  id: ID!
  name: String!
}
```

**Arguments:**

- `fields`: Represents a field selection set syntax.

## @shareable

```graphql
directive @shareable repeatable on OBJECT | FIELD_DEFINITION
```

By default, only a single source schema is allowed to contribute a particular
field to an object type. This prevents source schemas from inadvertently
defining similarly named fields that are not semantically equivalent.

```graphql counter-example
# Schema A
type Product {
  name: String!
  description: String!
}

# Schema B
type Product {
  name: String!
  variation: ProductVariation!
}
```

Fields must be explicitly marked as `@shareable` to allow multiple source
schemas to define them, ensuring that the decision to serve a field from more
than one source schema is intentional and coordinated.

```graphql example
# Schema A
type Product {
  name: String! @shareable
  description: String!
}

# Schema B
type Product {
  name: String! @shareable
  variation: ProductVariation!
}
```

If multiple source schemas define the same sharable field, they are assumed to
be semantically equivalent, and the executor is free to choose between them as
it sees fit.

The `@shareable` directive can also be applied at the object-type level, having
the same effect as if `@shareable` were applied to each field of the type.

```graphql example
# Schema A
type Product @shareable {
  name: String!
  description: String!
}

# Schema B
type Product {
  name: String! @shareable
  variation: ProductVariation!
}
```

Key fields of an object-type are considered shareable by default and do not need
to be explicitly marked with `@shareable`.

```graphql example
# Schema A
type Product @key(fields: "id") {
  id: ID!
  name: String! @shareable
  description: String!
}

# Schema B
type Product @key(fields: "id") {
  id: ID!
  name: String! @shareable
  variation: ProductVariation!
}
```

## @provides

```graphql
directive @provides(fields: FieldSelectionSet!) on FIELD_DEFINITION
```

The `@provides` directive indicates that a field can provide certain subfields
of its return type from the same source schema, without requiring an additional
resolution step elsewhere.

```graphql example
type Review {
  id: ID!
  body: String!
  author: User @provides(fields: "email")
}

type User @key(fields: "id") {
  id: ID!
  email: String! @external
  name: String!
}

type Query {
  reviews: [Review!]
  users: [User!]
}
```

When a field annotated with `@provides` returns an object, interface or union
type that may also be contributed by other source schemas, this directive
declares which of that type’s subfields the current source schema can resolve
directly.

```graphql example
{
  reviews {
    body
    author {
      name
      email
    }
  }
}
```

If a client tries to fetch the same subfield (`User.email`) through a different
path (e.g., users query field), the source schema will not be able to resolve it
and will throw an error.

```graphql counter-example
{
  users {
    # The source schema does NOT provide email in this context,
    # and this field will fail at runtime.
    email
  }
}
```

The `@provides` directive may reference multiple fields or nested fields:

```graphql example
type Review {
  id: ID!
  product: Product @provides(fields: "sku variation { size }")
}

type Product @key(fields: "sku variation { id }") {
  sku: String! @external
  variation: ProductVariation!
  name: String!
}

type ProductVariation {
  id: String!
  size: String! @external
}
```

When a field annotated with the provides directive has an abstract return type
the fields syntax can leverage inline fragments to express fields that can be
resolved locally.

```graphql example
type Review {
  id: ID!
  # The @provides directive tells us that this source schema can supply different
  # fields depending on which concrete type of Product is returned.
  product: Product
    @provides(
      fields: """
      ... on Book { author }
      ... on Clothing { size }
      """
    )
}

interface Product @key(fields: "id") {
  id: ID!
}

type Book implements Product {
  id: ID!
  title: String!
  author: String! @external
}

type Clothing implements Product {
  id: ID!
  name: String!
  size: String! @external
}

type Query {
  reviews: [Review!]!
}
```

The `@provides` directive is an execution-time optimization and never a
requirement for resolvability. Composition validates that every query path of
the composite schema remains satisfiable with all `@provides` directives
ignored. A `@provides` allows the _distributed GraphQL executor_ to obtain the
selected fields in the same response and thereby reduce the number of source
schema requests, but the selected fields must remain resolvable without it.

**Arguments:**

- `fields`: Represents a field selection set syntax describing the subfields of
  the returned type that can be provided by the current source schema.

## @external

```graphql
directive @external on FIELD_DEFINITION
```

The @external directive indicates that a field is recognized by the current
source schema but is not directly contributed (resolved) by it. Instead, this
schema references the field for specific composition purposes.

**Entity Keys**

When combined with one or more `@key` directives, an external field can serve as
an entity identifier (or part of a composite identifier).

```graphql example
type Query {
  productBySku(sku: String!): Product @lookup
  productByUpc(upc: String!): Product @lookup
}

type Product @key(fields: "sku") @key(fields: "upc") {
  sku: String! @external
  upc: String! @external
  name: String
}
```

**Field Resolution**

When another field in the same source schema uses `@provides` to declare that it
can resolve certain external fields in a single data-fetching step.

```graphql example
type Review {
  id: ID!
  text: String
  author: User @provides(fields: "email")
}

type User {
  id: ID!
  email: String! @external
}
```

When a field is marked `@external`, the composition process understands that the
field is provided by another source schema. The current source schema references
it only for entity identification (via `@key`) or for providing a field through
`@provides`. If no such usage exists, the presence of an `@external` field
produces a composition error.

The _distributed GraphQL executor_ never requests a field marked `@external`
from the declaring source schema directly. The field is resolved either by a
source schema that defines it without `@external`, or - when reached through a
field annotated with `@provides` - as part of the providing source schema's
response. The value of an external key field may also be known to the executor
without resolving the field, for example, when it was used as the input of a
lookup field that resolved the entity.

## @override

```graphql
directive @override(from: String!) on FIELD_DEFINITION
```

The `@override` directive is used to migrate a field from one source schema to
another. When a field in the local schema is annotated with
`@override(from: "Catalog")`, it signals that the local schema overrides the
field previously contributed by the `Catalog` source schema. As a result, the
composite schema will source this field from the local schema, rather than from
the original source schema.

The following example shows how a field can be migrated from the `Catalog`
schema to the new `Payments` schema. By using `@override`, a field can be moved
to a new schema without requiring any change to the original `Catalog` schema.

```graphql example
# The original "Catalog" schema:
type Product @key(fields: "id") {
  id: ID!
  name: String!
  price: Float!
}

# The new "Payments" schema:
type Product @key(fields: "id") {
  id: ID! @external
  price: Float! @override(from: "Catalog")
  tax: Float!
}
```

Fields that are annotated can themselves be migrated.

```graphql example
# The original "Catalog" schema:
type Product @key(fields: "id") {
  id: ID!
  name: String!
  price: Float!
}

# The new "Payments" schema:
type Product @key(fields: "id") {
  id: ID! @external
  price: Float! @override(from: "Catalog")
  tax: Float!
}

# The new "Pricing" schema:
type Product @key(fields: "id") {
  id: ID! @external
  price: Float! @override(from: "Payments")
  tax: Float!
}
```

If the composition detects cyclic overrides it must throw a composition error.

```graphql example
# The original "Catalog" schema:
type Product @key(fields: "id") {
  id: ID!
  name: String!
  price: Float! @override(from: "Pricing")
}

# The new "Payments" schema:
type Product @key(fields: "id") {
  id: ID! @external
  price: Float! @override(from: "Catalog")
  tax: Float!
}
```

**Arguments:**

- `from`: The name of the source schema that originally provided this field.

## @partial

```graphql
directive @partial(of: String!) on OBJECT
```

A _partial type_ contributes _default field implementations_ to an interface.
The fields of a partial type are declared once, keyed by the target interface's
`@key`, and composition projects them onto the interface and onto every object
type that implements it. The contributing source schema does not know or list
the concrete implementing types.

The `of` argument names the target interface, which must be declared in the same
source schema (see [Partial Target Invalid](#sec-Partial-Target-Invalid)). A
partial type never appears in the composite schema: it does not participate in
name-based type merging, and its fields reach the composite schema only through
projection.

In the following example, the `MediaReviews` partial type contributes the
`averageRating` field to the `Media` interface. Source schema A declares the
partial type and resolves it through the internal lookup field
`mediaReviewsById`; source schema B declares the concrete types that implement
`Media`.

```graphql example
# Source Schema A ("reviews")
interface Node {
  id: ID!
}

interface Media implements Node @key(fields: "id") {
  id: ID!
  title: String!
}

type MediaReviews @partial(of: "Media") @key(fields: "id") {
  averageRating(title: String! @require(field: "title")): Float!
}

type Query {
  mediaReviewsById(id: ID!): MediaReviews @lookup @internal
  topReviewed: [Media!]!
}

# Source Schema B ("catalog")
interface Node {
  id: ID!
}

interface Media implements Node @key(fields: "id") {
  id: ID!
  title: String!
}

type Book implements Media & Node @key(fields: "id") {
  id: ID!
  title: String!
  pages: Int!
}

type Photo implements Media & Node @key(fields: "id") {
  id: ID!
  title: String!
  averageRating: Float! @implement
}

type Query {
  mediaById(id: ID!): Media @lookup
  bookById(id: ID!): Book @lookup
}
```

The above example translates to the following composite schema. The
`averageRating` field is projected onto the `Media` interface and onto `Book`;
`Photo` provides its own implementation, which takes precedence (see
[@implement](#sec--implement)). The `MediaReviews` type does not exist in the
composite schema, and neither does the internal `mediaReviewsById` lookup field.
The `title` argument of `averageRating` is annotated with `@require` and is
therefore removed as well.

```graphql example
interface Node {
  id: ID!
}

interface Media implements Node {
  id: ID!
  title: String!
  averageRating: Float!
}

type Book implements Media & Node {
  id: ID!
  title: String!
  pages: Int!
  averageRating: Float!
}

type Photo implements Media & Node {
  id: ID!
  title: String!
  averageRating: Float!
}

type Query {
  mediaById(id: ID!): Media
  bookById(id: ID!): Book
  topReviewed: [Media!]!
}
```

A source schema that only contributes default fields stays small. The local
declaration of the target interface is required, but it only needs the key
fields that the partial type declares. It merges with the declarations of other
source schemas by the existing interface merge rules.

```graphql example
interface Media @key(fields: "id") {
  id: ID!
}

type MediaReviews @partial(of: "Media") @key(fields: "id") {
  averageRating: Float!
}

type Query {
  mediaReviewsById(id: ID!): MediaReviews @lookup @internal
}
```

Note: Fields declared on the local target-interface declaration merge into the
composite interface contract like any other interface declaration and thereby
obligate every implementing type to provide them. Declare on the local interface
only fields that genuinely belong to the shared contract.

**Declaring a Partial Type**

A partial type must declare at least one `@key` (see
[Partial Key Missing](#sec-Partial-Key-Missing)), and every `@key` on a partial
type must be identical to a `@key` declared on the target interface in the same
source schema (see [Partial Key Mismatch](#sec-Partial-Key-Mismatch)). The
source schema must also provide at least one lookup field that returns the
partial type (see [Partial Lookup Missing](#sec-Partial-Lookup-Missing)); the
_distributed GraphQL executor_ uses this lookup field to fetch the default
fields for an entity. The lookup field must be annotated with `@internal`: a
partial type never appears in the composite schema, so a public lookup field
returning it would reference a type that does not exist in the composite schema,
which composition rejects (see
[Reference To Internal Type](#sec-Reference-To-Internal-Type)). The lookup
remains available to the _distributed GraphQL executor_ for entity resolution.

A partial type must not declare `implements` (see
[Partial No Implements](#sec-Partial-No-Implements)), must not be referenced
anywhere except as the return type of lookup fields (see
[Partial Invalid Usage](#sec-Partial-Invalid-Usage)), and a source schema must
not declare more than one partial type for the same target interface (see
[Partial Duplicate Target](#sec-Partial-Duplicate-Target)).

**Effective Shape**

Field references in the `fields` argument of `@key` directives on a partial
type, and the argument mapping of lookup fields that return the partial type,
are resolved against the partial type's _effective shape_: the union of its own
fields and the fields of the target interface as declared in the same source
schema. This is why `@key(fields: "id")` is valid on `MediaReviews` in the
example above although the partial type does not declare an `id` field (see
[Key Invalid Fields](#sec-Key-Invalid-Fields)).

The selection map of a `@require` argument on a partial type's field is instead
rooted at the target interface: the required data is fetched from other source
schemas for entities of the target interface, so the partial type's own fields
are not selectable in a requirement (see
[Require Invalid Fields](#sec-Require-Invalid-Fields)).

**Explicit Intent**

Partial type semantics apply only when the `@partial` directive is present. An
object type that implements an interface and shares its key is a genuine
implementer of that interface; composition never reclassifies it as a partial
type, no matter how similar its shape is.

**Sharing Data via Requirements**

A default field takes the data it needs from other source schemas as arguments
annotated with `@require`, rooted at the target interface. As with any
requirement, these arguments are removed from the composite schema. A partial
type must not redeclare the fields of the target interface; the only fields of
the target interface it may declare are key fields (see
[Partial Field Duplicates Contract](#sec-Partial-Field-Duplicates-Contract)).

**Resolution**

The declared `@key` and the lookup field are the entire resolution mechanism:
the _distributed GraphQL executor_ resolves default fields by calling the
partial type's lookup field with the key values of the entity. The composite
schema treats this lookup as a lookup for the target interface that is not
authoritative for the concrete type: a result from it is an instance of the
target interface whose key is known but whose concrete type is unknown.

Because of this, results obtained through a partial type's lookup - or through
interface-typed fields that the contributing source schema resolves from its
partial data - never carry an authoritative concrete `__typename`. The
_distributed GraphQL executor_ never resolves `__typename`, inline fragments, or
type conditions through the partial type's lookup; concrete typing comes from a
lookup field that returns the target interface (see
[Default Typename Unresolvable](#sec-Default-Typename-Unresolvable) and
[Resolving Default Field Implementations](#sec-Resolving-Default-Field-Implementations)).
A source schema that also declares implementing types of the target interface
remains authoritative for those types through its ordinary lookup fields.

A source schema that contributes a partial type may itself expose fields typed
with the target interface, such as `topReviewed: [Media!]!` in the example
above. Executing such a field yields interface-typed results whose concrete type
is resolved on demand through another source schema's interface lookup.

**Migrating Fields into a Default**

A field on a partial type may carry `@override(from:)` to lift a field that is
resolved concretely in another source schema into a single default
implementation. Standard `@override` semantics apply. The opposite direction - a
single implementing type providing its own implementation - needs no
`@override`, because an implementer's own field always takes precedence over a
projected default (see [@implement](#sec--implement)).

Note: `@inaccessible` on a field of a partial type projects with the field,
hiding the projected default from the client-facing composite schema.
`@inaccessible` on the partial type itself has no effect, since the type never
enters the composite schema.

Note: This specification does not require a source schema to be executable on
its own. A server implementation MAY treat a partial type locally as a possible
type of its target interface; in the composite schema, the partial type is never
a possible type of the interface.

**Arguments:**

- `of`: The name of the target interface type.

## @implement

```graphql
directive @implement on FIELD_DEFINITION
```

When an implementing object type declares a field with the same name as a
projected default - in any source schema - the implementer's own field takes
precedence, and the default is not projected onto that type. This holds whether
or not the field is annotated with `@implement`, so introducing a new default
field is never a breaking change for existing implementing types.

The `@implement` directive is an optional marker that makes this intent
explicit: it declares that the field intentionally provides its own
implementation instead of taking the default. A field annotated with
`@implement` must shadow an existing default field (see
[Implement Without Default](#sec-Implement-Without-Default)); this catches stale
markers after a default has been removed. A field that shadows a default without
the marker composes successfully but is reported as a warning (see
[Default Field Shadowed](#sec-Default-Field-Shadowed)).

In the following example, `Photo` provides its own `averageRating`
implementation instead of the default contributed by the `MediaReviews` partial
type; implementing types without an own `averageRating` field take the default.

```graphql example
type Photo implements Media & Node @key(fields: "id") {
  id: ID!
  title: String!
  averageRating: Float! @implement
}
```
