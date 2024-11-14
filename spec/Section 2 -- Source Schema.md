# Source Schema

A _source schema_ is a GraphQL schema that is part of a larger _composite
schema_. Source schemas use directives to express intent and requirements for
the composition process. In the following chapters, we will describe the
directives that are used to annotate a source schema.

## Directives

### @lookup

```graphql
directive @lookup on FIELD_DEFINITION
```

The `@lookup` directive is used within a _source schema_ to specify output
fields that can be used by the _distributed GraphQL executor_ to resolve an
entity by a stable key.

The stable key is defined by the arguments of the field. Each argument must
match a field on the return type of the lookup field.

Source schemas can provide multiple lookup fields for the same entity that
resolve the entity by different keys.

In this example, the source schema specifies that the `Product` entity can be
resolved with the `productById` field or the `productByName` field. Both lookup
fields are able to resolve the `Product` entity but do so with different keys.

```graphql example
type Query {
  version: Int # NOT a lookup field.
  productById(id: ID!): Product @lookup
  productByName(name: String!): Product @lookup
}

type Product @key(fields: "id") @key(fields: "name") {
  id: ID!
  name: String!
}
```

The arguments of a lookup field must correspond to fields specified as an entity
key with the `@key` directive on the entity type.

```graphql example
type Query {
  node(id: ID!): Node @lookup
}

interface Node @key(fields: "id") {
  id: ID!
}
```

Lookup fields may return object, interface, or union types. In case a lookup
field returns an abstract type (interface type or union type), all possible
object types are considered entities and must have keys that correspond with the
field's argument signature.

```graphql example
type Query {
  product(id: ID!, categoryId: Int): Product @lookup
}

union Product = Electronics | Clothing

type Electronics @key(fields: "id categoryId") {
  id: ID!
  categoryId: Int
  name: String
  brand: String
  price: Float
}

type Clothing @key(fields: "id categoryId") {
  id: ID!
  categoryId: Int
  name: String
  size: String
  price: Float
}
```

The following example shows an invalid lookup field as the `Clothing` type does
not declare a key that corresponds with the lookup field's argument signature.

```graphql counter-example
type Query {
  product(id: ID!, categoryId: Int): Product @lookup
}

union Product = Electronics | Clothing

type Electronics @key(fields: "id categoryId") {
  id: ID!
  categoryId: Int
  name: String
  brand: String
  price: Float
}

# Clothing does not have a key that corresponds
# with the lookup field's argument signature.
type Clothing @key(fields: "id") {
  id: ID!
  categoryId: Int
  name: String
  size: String
  price: Float
}
```

If the lookup returns an interface, the interface must also be annotated with a
`@key` directive and declare its keys.

```graphql example
interface Node @key(fields: "id") {
  id: ID!
}
```

Lookup fields must be accessible from the Query type. If not directly on the
Query type, they must be accessible via fields that do not require arguments,
starting from the Query root type.

```graphql example
type Query {
  lookups: Lookups!
}

type Lookups {
  productById(id: ID!): Product @lookup
}

type Product @key(fields: "id") {
  id: ID!
}
```

Lookups can also be nested within other lookups and allow resolving nested
entities that are part of an aggregate. In the following example the `Product`
can be resolved by its ID but also the `ProductPrice` can be resolved by passing
in a composite key containing the product ID and region name of the product
price.

```graphql example
type Query {
  productById(id: ID!): Product @lookup
}

type Product @key(fields: "id") {
  id: ID!
  price(regionName: String!): ProductPrice @lookup
}

type ProductPrice @key(fields: "regionName product { id }") {
  regionName: String!
  product: Product
  value: Float!
}
```

Nested lookups must immediately follow the parent lookup and cannot be nested
with fields in between.

```graphql counter-example
type Query {
  productById(id: ID!): Product @lookup
}

type Product @key(fields: "id") {
  id: ID!
  details: ProductDetails
}

type ProductDetails {
  price(regionName: String!): ProductPrice @lookup
}

type ProductPrice @key(fields: "regionName product { id }") {
  regionName: String!
  product: Product
  value: Float!
}
```

### @internal

```graphql
directive @internal on FIELD_DEFINITION
```

The `@internal` directive is used to mark lookup fields as internal. Internal
lookup fields are not used as entry points in the composite schema and can only
be used by the _distributed GraphQL executor_ to resolve additional data for an
entity.

```graphql example
type Query {
  # lookup field and possible entry point
  reviewById(id: ID!): Review @lookup

  # internal lookup field
  productById(id: ID!): Product @lookup @internal
}
```

The `@internal` directive provides control over which source schemas are used to
resolve entities and which source schemas merely contribute data to entities.
Further, using `@internal` allows hiding "technical" lookup fields that are not
meant for the client-facing composite schema.

### @inaccessible

```graphql
directive @inaccessible on OBJECT | FIELD_DEFINITION
```

The `@inaccessible` directive is used to prevent specific objects or fields from
being accessible through the composite schema, even if they are accessible in
the underlying source schemas.

This directive is particularly useful for restricting access to fields or
objects that are either irrelevant to the client-facing API or sensitive in
nature, such as internal identifiers or fields intended only for backend use.

```graphql example
type Product @key(fields: "id") @key(fields: "sku") {
  id: ID!
  sku: String! @inaccessible
  internalNote: String
}

type Query {
  productById(id: ID!): Product
  productBySku(sku: String!): Product @inaccessible
}
```

The above example declares the key field `sku` to be inaccessible from
the composite schema. However, type system members marked as inaccessible can
still be used within the composite execution schema to fulfill requirements or
lookups.

In contrast to the `@internal` directive the `@inaccessible` directive hides a
type member from the composite schema even if other source schemas on the same type
system member have no `@inaccessible` directive.

```graphql example
# Source Schema A
type Product @key(fields: "id") @key(fields: "sku") {
  id: ID!
  sku: String! @inaccessible
  internalNote: String
}

# Source Schema B
type Product @key(fields: "sku") {
  sku: String!
  price: Float!
}

# Composite Schema
type Product {
  id: ID!
  internalNote: String
  price: Float!
}
```

The example below removes the field `internalProductById` and the type `Product`
from the the composite schema. However, type system members marked as
inaccessible can still be used within the composite execution schema to fulfil
requirements.

Unlike @internal, which restricts lookup fields to backend use, @inaccessible
completely excludes the marked elements from the composite schema.

```graphql example
type Product @inaccessible {
  id: ID!
  sku: String!
  internalNote: String
}

type Query {
  publicProductById(id: ID!): Product
  internalProductById(id: ID!): Product @inaccessible
}
```

### @is

```graphql
directive @is(field: FieldSelectionMap!) on ARGUMENT_DEFINITION
```

The `@is` directive is utilized on lookup fields to describe how the arguments
can be mapped from the entity type that the lookup field resolves. The mapping
establishes semantic equivalence between disparate type system members across
source schemas and is used in cases where the argument does not 1:1 align with a
field on the entity type.

In the following example, the directive specifies that the `id` argument on the
field `Query.personById` and the field `Person.id` on the return type of the
field are semantically the same.

Note: In this case the `@is` directive could also be omitted as the argument and
field names match.

```graphql example
extend type Query {
  personById(id: ID! @is(field: "id")): Person @lookup
}
```

The `@is` directive also allows referring to nested fields relative to `Person`.

```graphql example
extend type Query {
  personByAddressId(id: ID! @is(field: "address.id")): Person
}
```

The `@is` directive is not limited to a single argument.

```graphql example
extend type Query {
  personByAddressId(
    id: ID! @is(field: "address.id")
    kind: PersonKind @is(field: "kind")
  ): Person
}
```

The `@is` directive can also be used in combination with `@oneOf` to specify
lookup fields that can resolve entities by different keys.

```graphql example
extend type Query {
  person(
    by: PersonByInput @is(field: "{ id } | { addressId: address.id } { name }")
  ): Person
}

input PersonByInput @oneOf {
  id: ID
  addressId: ID
  name: String
}
```

**Arguments:**

- `field`: Represents a selection path map syntax.

### @require

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

The upper example would translate to the following in the _composite schema_.

```graphql example
type Product {
  id: ID!
  delivery(zip: String!): DeliveryEstimates
}
```

This can also be done by using input types. The selection path map specifies
which data is required and needs to be resolved from other source schemas. If
the input type is only used to express a requirements it is removed from the
composite schema.

```graphql example
type Product {
  id: ID!
  delivery(
    zip: String!
    dimension: ProductDimensionInput! @require(field: "{ size: dimension.size, weight: dimension.weight }"))
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
      @require(field: "{ productSize: dimension.size, productWeight: dimension.weight }"))
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

**Arguments:**

- `field`: Represents a selection path map syntax.

### @key

```graphql
directive @key(fields: SelectionSet!) repeatable on OBJECT | INTERFACE
```

The @key directive is used to designate an entity's unique key, which identifies
how to uniquely reference an instance of an entity across different source
schemas. It allows a source schema to indicate which fields form a unique
identifier, or **key**, for an entity.

```graphql example
type Product @key(fields: "id") {
  id: ID!
  sku: String!
  name: String!
  price: Float!
}
```

Each occurrence of the @key directive on an object or interface type specifies
one distinct unique key for that entity, which enables a gateway to perform
lookups and resolve instances of the entity based on that key.

```graphql example
type Product @key(fields: "id") @key(fields: "key") {
  id: ID!
  sku: String!
  name: String!
  price: Float!
}
```

While multiple keys define separate ways to reference the same entity based on
different sets of fields, a composite key allows to uniquely identify an entity
by using a combination of multiple fields.

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

**Arguments:**

- `fields`: Represents a selection set syntax.

### @shareable

```graphql
directive @shareable repeatable on OBJECT | FIELD_DEFINITION
```

By default, only a single source schema is allowed to contribute a particular
field to an object type. This prevents source schemas from inadvertently
defining similarly named fields that are semantically not the same.

Fields have to be explicitly marked as `@shareable` to allow multiple source
schemas to define it, and ensures that the step of allowing a field to be served
from multiple source schemas is an explicit, coordinated decision.

If multiple source schemas define the same field, these are assumed to be
semantically equivalent, and the executor is free to choose between them as it
sees fit.

Note: Key fields are always considered sharable.

### @provides

```graphql
directive @provides(fields: SelectionSet!) on FIELD_DEFINITION
```

The `@provides` directive is an optimization hint specifying child fields that
can be resolved locally at the given source schema through a particular query
path. This allows for a variation of overlapping field to improve data fetching.

**Arguments:**

- `fields`: Represents a selection set syntax.

### @external

```graphql
directive @external on OBJECT_DEFINITION | INTERFACE_DEFINITION | FIELD_DEFINITION
```

The `@external` directive is used in combination with the `@provides` directive
and specifies data that is not owned ba a particular source schema.

### @override

```graphql
directive @override(from: String!) on FIELD_DEFINITION
```

The `@override` directive allows to migrate fields from one source schema to
another.
