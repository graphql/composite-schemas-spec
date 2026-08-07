# Source Schema

A _source schema_ is a GraphQL schema that is part of a larger _composite
schema_. Source schemas use directives to express intent and requirements for
the composition process as well as to describe runtime behavior. The following
chapters describe the directives that are used to annotate a source schema.

## Entities and Identity

An _entity_ is a type whose instances have an _identity_ that is stable across
_source schemas_, allowing the _distributed GraphQL executor_ to recognize that
data contributed by different source schemas describes the same object. An
_entity_ has one or more _stable keys_ that represent its _identity_. A _stable
key_ is a set of one or more fields that represents the _identity_ of an
_entity_ for comparison.

The _identity_ of an _entity_ serves two purposes: comparison and recall.
Comparison recognizes that two instances refer to the same _entity_. Recall
fetches an _entity_ again by one of its _stable keys_.

The `@key` directive provides the declarative _identity_ and comparison half. It
declares that a type is an _entity_ and identifies which field set or field sets
are its _stable keys_. Declaring _identity_ locally on the type keeps it visible
without requiring a reader or tool to scan every lookup field across every
_source schema_ to infer what identifies the _entity_. This follows the
Explicitness design principle and supports the Collaborative design principle by
surfacing _identity_ where source schemas coordinate on a shared type.

The `@lookup` directive provides the recall half. It lets the _distributed
GraphQL executor_ resolve an _entity_ by a _stable key_ in a _source schema_.

In the following example, the `Product` type declares its _identity_ with
`@key(fields: "id")`. The `productById` lookup field provides recall for the
same _stable key_.

```graphql example
type Product @key(fields: "id") {
  id: ID!
  name: String!
  price: Float!
}

type Query {
  productById(id: ID!): Product @lookup
}
```

A type MAY declare a `@key` for which no `@lookup` field exists in any _source
schema_; such a _stable key_ represents _identity_ and supports comparison but
cannot be used by the _distributed GraphQL executor_ to resolve, or recall, the
_entity_.

In the following example, `sku` is a _stable key_ for comparison. No lookup
field resolves `Product` by `sku` in any _source schema_.

```graphql example
type Product @key(fields: "id") @key(fields: "sku") {
  id: ID!
  sku: String!
}

type Query {
  productById(id: ID!): Product @lookup
}
```

A `@key` that could be inferred from a lookup field's arguments MAY be omitted.
A type that can be resolved by a _stable key_ SHOULD declare a corresponding
`@key` for that _stable key_, even though the _stable key_ could be inferred
from the lookup field's arguments. This double bookkeeping keeps the _identity_
of an _entity_ explicit and locally visible on the type rather than scattered
across the lookup fields of every _source schema_, in keeping with the
Explicitness and Collaborative design principles.

Note: A single identifier used both to compare and to fetch an _entity_ couples
two independent concerns. Separating the comparison _stable key_ from the lookup
mechanism lets comparison use small, stable values and avoids bloated keys and
expensive comparisons.

## @lookup

```graphql
directive @lookup on FIELD_DEFINITION
```

The `@lookup` directive is used within a _source schema_ to specify output
fields that can be used by the _distributed GraphQL executor_ to resolve an
_entity_ by a _stable key_.

For a lookup field, the _stable key_ used for recall is represented by the
arguments of the field. Each lookup argument must match a field on the return
type of the lookup field. The matched field does not need to be defined in the
source schema that declares the lookup field; it must exist on the return type
in at least one source schema. The _distributed GraphQL executor_ resolves the
key value from the source schemas where the field is available.

Source schemas can provide multiple lookup fields for the same _entity_ to
resolve the _entity_ by different _stable keys_.

In this example, the source schema specifies that the `Product` _entity_ can be
resolved with the `productById` field or the `productByName` field. Both lookup
fields are able to resolve the `Product` _entity_ but do so with different
_stable keys_.

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
object types of the abstract return type are considered _entities_, and each
must have fields that correspond to every argument of the lookup field. When an
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
fields for resolving _entities_ that should not be accessible through the
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

Internal fields can only be used by the _distributed GraphQL executor_ as lookup
fields for _entity_ resolution.

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
can be mapped from the _entity_ type that the lookup field resolves. The mapping
establishes semantic equivalence between disparate type system members across
source schemas and is used in cases where an argument does not directly align
with a field on the _entity_ type.

An `@is` selection map must not supply arguments; the mapping must consist of
plain field paths. The arguments of a lookup field represent a _stable key_ of
the _entity_, and a _stable key_ must map to plain field values. A referenced
field may still declare arguments, as long as each argument is nullable, has a
default value, or is annotated with `@require`, so that the field can be
resolved without any arguments being supplied.

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
single lookup field that can resolve _entities_ by multiple _stable keys_.

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

The `@require` directive can also be applied to arguments on interface fields.
The selection map is rooted at the interface type and is evaluated against the
concrete runtime object. The annotation must be applied consistently on the
interface field and on the corresponding argument of every implementing field.
The selection maps themselves may differ, and an implementing type may derive
the required value from implementation-specific fields.

```graphql example
interface Account {
  id: ID!
  preferredLocale: String
  displayName(locale: String @require(field: "preferredLocale")): String
}

type User implements Account {
  id: ID!
  preferredLocale: String
  displayName(locale: String @require(field: "preferredLocale")): String
}

type Organization implements Account {
  id: ID!
  preferredLocale: String
  billingLocale: String
  displayName(locale: String @require(field: "billingLocale")): String
}
```

Since the annotation is consistent, composition removes the `locale` argument
from the interface field and from all implementing fields together, keeping the
interface contract of the _composite schema_ intact.

```graphql example
interface Account {
  id: ID!
  preferredLocale: String
  displayName: String
}

type User implements Account {
  id: ID!
  preferredLocale: String
  displayName: String
}

type Organization implements Account {
  id: ID!
  preferredLocale: String
  billingLocale: String
  displayName: String
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

The `@require` directive must not be used on arguments of fields annotated with
`@lookup`. The arguments of a lookup field represent the stable key with which
the _distributed executor_ resolves an entity; they are supplied from an
existing representation of the entity, and a requirement has no defined meaning
in that position.

**Arguments:**

- `field`: Represents a selection path map syntax.

## @key

```graphql
directive @key(fields: FieldSelectionSet!) repeatable on OBJECT | INTERFACE
```

The `@key` directive is used to designate a _stable key_ of an _entity_, which
identifies how to uniquely reference an instance of an _entity_ across different
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
one distinct _stable key_ for that _entity_. These _stable keys_ allow the
_distributed GraphQL executor_ to distinguish between different _entities_ of
the same type.

```graphql example
type Product @key(fields: "id") @key(fields: "sku") {
  id: ID!
  sku: String!
  name: String!
  price: Float!
}
```

While multiple _stable keys_ define separate ways to reference the same _entity_
based on different sets of fields, a composite _stable key_ allows for uniquely
identifying an _entity_ by using a combination of multiple fields.

```graphql example
type Product @key(fields: "id sku") {
  id: ID!
  sku: String!
  name: String!
  price: Float!
}
```

The directive is applicable to both OBJECT and INTERFACE types. This allows
_entities_ that implement an interface to inherit the _stable keys_ defined at
the interface level, ensuring consistent identification across different
implementations of that interface.

By applying the `@key` directive all referenced fields become sharable even if
the fields are not explicitly marked with `@shareable`.

```graphql example
# Source Schema A
type Product @key(fields: "id") {
  id: ID!
  price: Float!
}

# Source Schema B
type Product @key(fields: "id") {
  id: ID!
  name: String!
}
```

Fields must be explicitly marked as part of a _stable key_ or annotated with the
`@shareable` directive to allow multiple source schemas to define them, ensuring
that the decision to serve a field from more than one source schema is
intentional and coordinated.

```graphql counter-example
# Source Schema A
type Product @key(fields: "id") {
  id: ID!
  price: Float!
}

# Source Schema B
type Product {
  id: ID!
  name: String!
}
```

**Arguments:**

- `fields`: Represents a field selection set syntax.

## @interfaceObject

```graphql
directive @interfaceObject on OBJECT
```

The `@interfaceObject` directive is used within a source schema to declare an
object type that acts as a _stand-in_ for an interface defined in another source
schema. The stand-in carries the same name as the interface and allows the
source schema to contribute fields to the interface without defining its
implementing types.

```graphql example
type Media @interfaceObject @key(fields: "id") {
  id: ID!
  reviews: [Review!]!
}
```

Composition merges the stand-in into the interface instead of reporting a
type-kind conflict. If no source schema defines the interface, composition fails
with an error.

In the following example, source schema A defines the `Media` interface. Source
schema B defines a stand-in for `Media`.

```graphql example
# Source Schema A
interface Media @key(fields: "id") {
  id: ID!
  title: String!
}

# Source Schema B
type Media @interfaceObject @key(fields: "id") {
  id: ID!
  reviews: [Review!]!
}

type Review {
  rating: Int!
}

# Composite Schema
interface Media {
  id: ID!
  title: String!
  reviews: [Review!]!
}

type Review {
  rating: Int!
}
```

Composition adds each stand-in field that is not part of a key to the interface
and projects its implementation onto every type that implements the interface.

In the following example, composition projects the implementation of `reviews`
from the `Media` stand-in onto `Book`, although no source schema declares the
field directly on `Book`.

```graphql example
# Source Schema A
interface Media @key(fields: "id") {
  id: ID!
  title: String!
}

type Book implements Media @key(fields: "id") {
  id: ID!
  title: String!
}

# Source Schema B
type Media @interfaceObject @key(fields: "id") {
  id: ID!
  reviews: [Review!]!
}

type Review {
  rating: Int!
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
  reviews: [Review!]!
}

type Review {
  rating: Int!
}
```

A stand-in must declare a `@key` that matches one of the keys declared on the
interface.

A stand-in is not required to declare a lookup field. Without one, each non-key
field must either be `@shareable` with another effective owner reachable from
every context that needs the field, or be replaced with `@implement` on every
implementing type. A stand-in that declares only key fields serves as a typed
reference to the entity.

```graphql example
# Source Schema A
type Media @interfaceObject @key(fields: "id") {
  id: ID!
}

type Rating {
  id: ID!
  subject: Media!
  stars: Int!
}

# Source Schema B
type Query {
  mediaById(id: ID!): Media @lookup
}

interface Media @key(fields: "id") {
  id: ID!
}

type Book implements Media {
  id: ID!
}
```

## @implement

```graphql
directive @implement on FIELD_DEFINITION
```

A stand-in contributes field implementations that composition projects onto an
interface's implementing types. The `@implement` directive signals the intent to
replace such a projected implementation with an explicit implementation on the
annotated field.

An object type that implements an interface may provide an explicit
implementation by declaring the field itself and marking it with `@implement`.
The type's own field is then used instead of the implementation projected from
the interface object. The directive controls precedence between the type's own
field and a less-specific projected implementation. It does not require that the
type's field have only one resolver: multiple explicit implementations may
remain eligible when they are also marked with `@shareable`.

A direct implementation may instead coexist with an applicable projected
implementation when every declaration is marked with `@shareable`. In that case,
the projected implementation is not replaced. Both the stand-in schema and the
schema that declares the field directly remain eligible to resolve it. If the
declarations are neither an explicit replacement nor all shareable, composition
fails because the collision is ambiguous.

The directives answer independent questions:

- `@implement` determines whether a more-specific declaration replaces an
  applicable, less-specific projected implementation.
- `@shareable` determines whether multiple declarations that remain after that
  precedence decision are interchangeable.

Consequently, `@shareable` without `@implement` preserves both the projected and
direct implementations. `@implement` removes the projected implementation. When
several source schemas provide that explicit implementation, they use both
`@implement` and `@shareable`: the former replaces the projected implementation,
and the latter keeps the explicit implementations interchangeable.

In the following example, composition projects the `taxRate` implementation from
source schema B's stand-in onto every `Product`. `Chair` provides an explicit
implementation of `taxRate`.

```graphql example
# Source Schema A
interface Product @key(fields: "id") {
  id: ID!
  name: String!
}

type Chair implements Product @key(fields: "id") {
  id: ID!
  name: String!
  taxRate: Float! @implement
}

# Source Schema B
type Product @interfaceObject @key(fields: "id") {
  id: ID!
  taxRate: Float!
}
```

Without `@implement`, composition rejects `Chair.taxRate` in this example,
because neither declaration is `@shareable`. Marking both declarations with
`@shareable` would instead preserve both as eligible implementations.

Multiple source schemas may provide the explicit implementation at the same
specificity level. Each declaration uses `@implement` to replace the
less-specific projected implementation and `@shareable` to remain
interchangeable with the other explicit implementations.

```graphql example
# Source Schema A
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
type Product @interfaceObject @key(fields: "id") {
  id: ID!
  taxRate: Float @shareable
}

# Source Schema C
type Chair @key(fields: "id") {
  id: ID!
  taxRate: Float @implement @shareable
}

# Source Schema D
type Chair @key(fields: "id") {
  id: ID!
  taxRate: Float @implement @shareable
}
```

For `Chair.taxRate`, source schemas C and D are the eligible implementations;
source schema B's less-specific projected implementation is replaced. Source
schema B remains the eligible implementation of `Table.taxRate`.

In an interface hierarchy, a stand-in for a more specific interface may also
provide an explicit implementation. Its projected implementation then replaces
the implementation projected from the less-specific interface for every type
that implements the more-specific interface.

In the following example, source schema B provides a projected `taxRate`
implementation for every `Product`. Source schema C provides a more-specific
projected implementation of `taxRate` for every `PhysicalProduct`.

```graphql example
# Source Schema A
interface Product @key(fields: "id") {
  id: ID!
}

interface PhysicalProduct implements Product @key(fields: "id") {
  id: ID!
  weight: Float!
}

type Chair implements PhysicalProduct & Product @key(fields: "id") {
  id: ID!
  weight: Float!
}

type Ebook implements Product @key(fields: "id") {
  id: ID!
}

# Source Schema B
type Product @interfaceObject @key(fields: "id") {
  id: ID!
  taxRate: Float!
}

# Source Schema C
type PhysicalProduct @interfaceObject @key(fields: "id") {
  id: ID!
  taxRate: Float! @implement
}
```

`Chair` implements `PhysicalProduct`, so source schema C's `taxRate`
implementation is projected onto it. `Ebook` implements only `Product`, so
source schema B's implementation is projected onto it. Without `@implement` on
`PhysicalProduct.taxRate`, composition rejects the schema because the two
projected implementations collide. The declarations could instead both use
`@shareable`, in which case both implementations would remain eligible rather
than one replacing the other.

If no matching projected implementation exists for a field marked with
`@implement`, composition fails. The same applies to `@implement` on an
interface field, since an interface field is a contract declaration and cannot
replace a projected implementation.

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

This also applies when one declaration is projected from an `@interfaceObject`
stand-in and another is declared directly on an implementing type. If both
declarations are `@shareable`, both remain eligible. If the implementing type
uses `@implement`, the projected declaration is replaced instead. The directives
express independent properties: `@implement` controls precedence over a
less-specific projected implementation, while `@shareable` permits multiple
eligible declarations at the resulting level.

In the following example, source schema B provides a projected implementation of
`taxRate`, while source schema C declares the same field directly on `Chair`.
Both declarations are `@shareable`, and neither uses `@implement`, so both
remain eligible for `Chair.taxRate`.

```graphql example
# Source Schema A
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
type Product @interfaceObject @key(fields: "id") {
  id: ID!
  taxRate: Float @shareable
}

# Source Schema C
type Chair @key(fields: "id") {
  id: ID!
  taxRate: Float @shareable
}
```

The executor may resolve `Chair.taxRate` through source schema B or source
schema C. `Table` has no direct declaration, so source schema B remains its only
eligible implementation of `taxRate`.

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
ignored. A `@provides` directive allows the _distributed GraphQL executor_ to
obtain the selected fields in the same response and thereby reduce the number of
source schema requests, but the selected fields must remain resolvable without
it.

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

**Stable Keys**

When combined with one or more `@key` directives, an external field can serve as
a _stable key_ (or part of a composite _stable key_).

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
it only for _entity_ identification (via `@key`) or for providing a field
through `@provides`. If no such usage exists, the presence of an `@external`
field produces a composition error.

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

The `@override` directive may also be applied to a field on an
`@interfaceObject` stand-in. Before composition projects stand-in field
implementations, it drops every declaration of the field, in the source schema
named by `from`, across the target interface's whole implementation closure: on
every implementing type, on every more-specific interface's stand-in, and even
on the source schema's own stand-in for the same interface. Dropping the source
schema's own stand-in declaration is what moves the projected implementation
from one schema to another.

As with any other use of `@override`, `from` names exactly one source schema.
Composition rejects cyclic overrides on stand-in fields, as it does for any
other field's `@override`. A dead override, one whose `from` schema declares no
matching field, composes normally. Nothing is dropped in that case.

A field projected from a stand-in is not itself a declaration for the purposes
of `@override`. An implementing type cannot use `@override` to replace a
projected implementation; `@implement` does that instead. A type that acquires a
field through `@override` becomes a direct declarer of the field from that point
on. When that direct declaration collides with an implementation projected from
an `@interfaceObject`, it must use `@implement` to replace the projected
implementation. Alternatively, both declarations may use `@shareable`, allowing
both schemas to resolve the field.

In the following example, the `Catalog` schema originally contributes `reviews`
directly on `Book` and `Movie`. The `Reviews` schema takes over by declaring
`Media` as a stand-in and overriding the field from `Catalog`.

```graphql example
# The original "Catalog" schema:
interface Media @key(fields: "id") {
  id: ID!
  title: String!
}

type Book implements Media @key(fields: "id") {
  id: ID!
  title: String!
  author: String!
  reviews: [Review!]!
}

type Movie implements Media @key(fields: "id") {
  id: ID!
  title: String!
  director: String!
  reviews: [Review!]!
}

type Review {
  id: ID! @shareable
  rating: Int! @shareable
}

# The new "Reviews" schema:
type Media @interfaceObject @key(fields: "id") {
  id: ID!
  reviews: [Review!]! @override(from: "Catalog")
}

type Review {
  id: ID! @shareable
  rating: Int! @shareable
}
```

Composition drops `Book.reviews` and `Movie.reviews` from the `Catalog` schema.
It instead projects `reviews` onto `Media`, and from there onto `Book` and
`Movie`, using the implementation contributed by the `Reviews` stand-in. The
composite schema is unchanged; only the source of the field moves.

**Arguments:**

- `from`: The name of the source schema that originally provided this field.
