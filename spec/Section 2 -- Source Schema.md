# Source Schema

## Directives

### @lookup

```graphql
directive @lookup(map: SelectionPath) on FIELD_DEFINITION
```

The `@lookup` directive is used within a _source schema_ to specify object
fields that can be used by the _distributed GraphQL executor_ to resolve an
entity by a stable key.

The stable key is defined by the arguments of the field. Only fields that are
annotated with the `@lookup` directive will be recognized as lookup field.

Source schemas can provide multiple lookup fields for the same entity with
different keys.

In this example the source schema specifies that the `Person` entity can be
resolved with the `personById` field or the `personByName` field on the `Query`
type. Both fields can resolve the same entity but do so with different keys.

```graphql example
type Query {
  version: Int # NOT a lookup field.
  personById(id: ID!): Person @lookup(map: "{ id: id }")
  personByName(name: String!): Person @lookup(map: "{ name: name }")
}

type Person @key(fields "id") @key(fields "name") {
  id: ID!
  name: String!
}
```

The selection syntax provided as a value to the `map` argument of the `@lookup`
directive must correspond to the all arguments of a lookup field. Further it
must correspond to fields specified by a `@key` directive annotated on the
return type of the lookup field.

```graphql example
type Query {
  node(id: ID!): Node @lookup(map: "{ id: id }")
}

interface Node @key(fields "id")  {
  id: ID!
}
```

Lookup fields may return object, interface or union types. In case a lookup
field returns an interface or union type all possible object types are
considered entities and must have keys that correspond with the lookup map.

```graphql example
type Query {
  entityById(id: ID!, categoryId: Int): Entity @lookup(map: "{ id: id, categoryId: categoryId }")
}

union Entity = Cat | Dog

type Dog @key(fields "id categoryId") {
  id: ID!
  categoryId: Int
}

type Cat @key(fields "id categoryId") {
  id: ID!
  categoryId: Int
}
```

The following example shows an invalid lookup field as the `Cat` type does not
declare a key that corresponds with the lookup fields argument signature.

```graphql counter-example
type Query {
  entityById(id: ID!, categoryId: Int): Entity @lookup(map: "{ id: id, categoryId: categoryId }")
}

union Entity = Cat | Dog

type Dog @key(fields "id categoryId") {
  id: ID!
  categoryId: Int
}

type Cat @key(fields "id") {
  id: ID!
}
```

If the lookup returns an interface in particular, the interface must also be
annotated with a `@key` directive.

```graphql example
interface Node @key(fields "id")  {
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
  personById(id: ID!): Person @lookup(map: "{ id: id }")
}

type Person @key(fields "id") {
  id: ID!
}
```

Lookup fields can also use the `@oneOf` directive to specify a lookup field that
can resolve multiple keys.

```graphql example
type Query {
  person(finder: PersonFinderInput!): Person @lookup(map: "{ name: name } | { id: id }")
}

type Person @key(fields "id") @key(fields "name") {
  id: ID!
  name: String!
}

input PersonFinderInput @oneOf {
  id: ID
  name: String
}
```

**Arguments:**

- `map`: Represents a selection path that describes how keys are mapped to
  arguments of a lookup field.

### @patch

```graphql
directive @patch(map: SelectionPath) on FIELD_DEFINITION
```

The `@patch` directive is used within a _source schema_ to specify object fields
that can be used by the _distributed GraphQL executor_ to resolve additional
data for an entity rather than fetching the entity itself. A patch resolver
result does noth mean that the actual entity exists.

```graphql example
type Query {
  personById(id: ID!): Person @patch(map: "{ id: id }")
  personByName(name: String!): Person @patch(map: "{ name: name }")
}

type Person @key(fields "id") @key(fields "name") {
  id: ID!
  name: String!
}
```

Patch resolve as oposed to lookup fields will be omitted from the _composite
schema_ but will be referenced within the _composite execution schema_.

**Arguments:**

- `map`: Represents a selection path that describes how keys are mapped to
  arguments of a lookup field.

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
