# Source Schema

## Directives

### @lookup

```graphql
directive @lookup on FIELD_DEFINITION
```

The `@lookup` directive is used within a _source schema_ to specify output
fields that can be used by the _distributed GraphQL executor_ to resolve an
entity by a stable key.

The stable key is defined by the arguments of the field. Only fields that are
annotated with the `@lookup` directive will be recognized as lookup fields.

Source schemas can provide multiple lookup fields for the same entity with
different sets of keys.

In this example, the source schema specifies that the `Product` entity can be
resolved with the `productById` field or the `productByName` field on the
`Query` type. Both fields can resolve the same entity but do so with different
keys.

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

The arguments of a lookup field must correspond to fields specified by a `@key`
directive annotated on the return type of the lookup field.

```graphql example
type Query {
  node(id: ID!): Node @lookup
}

interface Node @key(fields: "id") {
  id: ID!
}
```

Lookup fields may return object, interface, or union types. In case a lookup
field returns an interface or union type, all possible object types are
considered entities and must have keys that correspond with the fields argument
signature.

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

type Clothing @key(fields: "id") {
  id: ID!
  categoryId: Int
  name: String
  size: String
  price: Float
}
```

If the lookup returns an interface, the interface must also be annotated with a
`@key` directive.

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

### @internal

```graphql
directive @internal on FIELD_DEFINITION
```

-- make it more clerar that it only hides this for the subgraph it is annotated with internal
-- rethink firs use-case ... is it really needed?
-- only used in combination with lookup

The `@internal` directive signals to the composition process that annotated
field definitions are not intended to be part of the public schema. Internal
field definitions can still be used by the distributed GraphQL executor to
resolve data.

```graphql example
type Product @key(fields: "_internalId") {
  _internalId: String @internal
}
```

The `@internal` directive in combination with the `@lookup` directive allows
defining lookup directives that are not used as global fields in the public
schema and thus are not used as entry points.

```graphql example
type Query {
  # lookup field and possible entry point
  reviewById(id: ID!): Review @lookup

  # internal lookup field
  productById(id: ID!): Product @lookup @internal
}
```

This provides control over which source schemas are used to resolve entities and
which merely provide data to entities. It also allows hiding "technical" lookup
fields from the public schema.

### @is

```graphql
directive @is(map: FieldSelectionMap!) on ARGUMENT_DEFINITION
```

The `@is` directive is utilized in a lookup field to establish how the

semantic equivalence between disparate type system members across distinct
subgraphs, which the schema composition uses to connect types.

In the following example, the directive specifies that the `id` argument on the
field `Query.personById` and the field `Person.id` on the return type of the
field are semantically the same. This information is used to infer an entity
resolver for `Person` from the field `Query.personById`.

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

**Arguments:**

- `field`: Represents a selection path syntax.

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
    size: Int! @require(field: "dimension.size")
    weight: Int! @require(field: "dimension.weight")
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

If the input types do not match it can also be mapped with the path selection
syntax.

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
