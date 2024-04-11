# Source Schema

## Directives

### @lookup

```graphql
directive @lookup on FIELD_DEFINITION
```

Lookup fields allow the composite schema to resolve entities from a source schema by a stable key. The stable key is defined by the arguments of the field. Only fields that are annotated with the `@lookup` directive will be recognized as lookup field.

```graphql example
extend type Query {
  version: Int # NOT an entity resolver.
  personById(id: ID!): Person @lookup
}

extend type Person @key(fields "id") {
  id: ID!
}
```

The arguments of a lookup field must match fields specified with a `@key` directive annotated on the return type for the lookup field.

```graphql example
extend type Query {
  node(id: ID!): Node @lookup
}

interface Node @key(fields "id")  {
  id: ID!
}
```

Lookup fields returning an interface type can be used as lookup field for all implementing object types.

```graphql example
extend type Query {
  entityById(id: ID!, categoryId: Int): Entity @lookup
}

union Entity = Cat | Dog

extend type Dog {
  id: ID!
  categoryId: Int
}

extend type Cat {
  id: ID!
  categoryId: Int
}
```

....

```graphql example
extend type Query {
  lookups: Lookups!
}

type Lookups {
  personById(id: ID!): Person @lookup
}

extend type Person {
  id: ID! # matches the argument of personById
}
```

### @is

```graphql
directive @is(
  field: FieldSelection
  coordinate: Schemacoordinate
) on FIELD_DEFINITION | ARGUMENT_DEFINITION | INPUT_FIELD_DEFINITION
```

The `@is` directive is utilized to establish semantic equivalence between
disparate type system members across distinct subgraphs, which the schema
composition uses to connect types.

In the following example, the directive specifies that the `id` argument on the
field `Query.personById` and the field `Person.id` on the return type of the
field are semantically the same. This information is used to infer an entity
resolver for `Person` from the field `Query.personById`.

```graphql example
extend type Query {
  personById(id: ID! @is(field: "id")): Person @entityResolver
}
```

The `@is` directive also allows to refer to nested fields relative to `Person`.

```graphql example
extend type Query {
  personByAddressId(id: ID! @is(field: "address { id }")): Person
}
```

The `@is` directive not limited to a single argument.

```graphql example
extend type Query {
  personByAddressId(
    id: ID! @is(field: "address { id }")
    kind: PersonKind @is(field: "kind")
  ): Person
}
```

The directive can also establish semantic equivalence between two output fields.
In this example, the field `productSKU` is semantically equivalent to the field
`Product.sku`, allowing the schema composition to infer the connection of the
`Product` with the `Review` type.

```graphql example
extend type Review {
  productSKU: ID! @is(coordinate: "Product.sku") @internal
  product: Product @resolve
}
```

The `@is` directive can use either the `field` or `coordinate` argument. If both
are specified, the schema composition must fail.

```graphql counter-example
extend type Review {
  productSKU: ID!
    @is(coordinate: "Product.sku", field: "product { sku }")
    @internal
  product: Product @resolve
}
```

**Arguments:**

- `field`: Represents a GraphQL field selection syntax that refers to field
  relative to the current type; or, when used on arguments it refers to a field
  relative to the return type.
- `coordinate`: Represents a schema coordinate that refers to a type system
  member.

### @shareable

```graphql
directive @shareable repeatable on OBJECT | FIELD_DEFINITION
```

By default, only one subgraph is allowed to contribute a particular field to an
object type. This prevents subgraphs from inadvertently defining similarly named
fields that are semantically not the same.

Fields have to be explicitly marked as `@shareable` to allow multiple subgraphs
to define it. And it ensures the step of allowing a field to be served from
multiple subgraphs is an explicit, coordinated decision.

If multiple subgraphs define the same field, these are assumed to be
semantically equivalent, and the executor is free to choose between them as it
sees fit.

Note: Key fields are always considered sharable.

### @require

```graphql
directive @require(
  field: FieldSelection!
) on ARGUMENT_DEFINITION | INPUT_FIELD_DEFINITION
```

The `@require` directive is used to express data requirements with other
subgraphs. Arguments annotated with the `@require` directive are removed from
the public exposed schema and the value for these will be resolved by the
executor.

```graphql example
type Product {
  id: ID!
  delivery(
    zip: String!
    size: Int! @require(field: "dimension { size }")
    weight: Int! @require(field: "dimension { weight }")
  ): DeliveryEstimates
}
```

This can also be done by using input types. All fields of the input type that
match the required output type are required. If the input type is only used to
express a requirement it is removed from the public schema.

```graphql example
type Product {
  id: ID!
  delivery(
    zip: String!
    dimension: ProductDimensionInput! @require(field: "dimension"))
  ): DeliveryEstimates
}
```

### @provides

```graphql
directive @provides(fields: SelectionSet!) on FIELD_DEFINITION
```

The `@provides` directive is an optimization hint specifying child fields that
can be resolved locally at the given subgraph through a particular query path.
This allows for a variation of overlapping field to improve data fetching.

### @external

```graphql
directive @external on OBJECT_DEFINITION | INTERFACE_DEFINITION | FIELD_DEFINITION
```

The `@external` directive is used in combination with the `@provides` directive
and specifies data that is not owned ba a particular subgraph.

### @override

```graphql
directive @override(from: String!) on FIELD_DEFINITION
```

The `@override` directive allows to migrate fields from one subgraph to another.

### @internal

```graphql
directive @internal on OBJECT | INTERFACE | FIELD_DEFINITION | UNION | ENUM | ENUM_VALUE | INPUT_OBJECT | INPUT_FIELD_DEFINITION | SCALAR
```

The `@internal` directive signals to the composition process that annotated type
system members shall not be included into the public schema but still can be
used by the executor to build resolvers.

### Schemacoordinate

```graphql
scalar Schemacoordinate
```

The `Schemacoordinate` scalar represents a schema coordinate syntax.

```graphql example
Product.id
```

```graphql example
Product.estimateDelivery(zip:)
```
