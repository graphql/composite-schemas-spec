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

Lookups, can also be nested with other lookups and allow resolving nested
entities that are part of an aggregate. In the following example the `Product`
can be resolved by it's id but also the `ProductPrice` can be resolved by
passing in a composite key containing the product id and region name of the
product price.

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

Nested lookups must imediately follow the parent lookup and cannot be nested
with fields inbetween.

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

The `@is` directive can also be used in combination with oneOf to specify lookup
fields that can resolve entities by different keys.

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

### @shareable

```graphql
directive @shareable repeatable on OBJECT | FIELD_DEFINITION
```

By default, only one source schema is allowed to contribute a particular field
to an object type. This prevents source schemas from inadvertently defining
similarly named fields that are semantically not the same.

Fields have to be explicitly marked as `@shareable` to allow multiple source
schemas to define it. And it ensures the step of allowing a field to be served
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
