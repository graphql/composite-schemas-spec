# Directives

## Shared

In this section we outline directives and types that are shared between the subgraph configuration and the gateway configuration document.

### @fusion

```graphql
directive @fusion(
  prefix: String
  prefixSelf: Boolean! = false
  version: Version!
) on SCHEMA
```

The `@fusion` directive is applied to a schema definition node or a schema extension node in a GraphQL schema document and defines the Fusion specification version that the configuration document follows. If the version is not explicitly stated, Fusion tooling is expected to select the newest specification version the tooling follows.

```graphql example
schema @fusion(version: "2023-12") {
  query: Query
}
```

In addition, it specifies a prefix for Fusion-specific types and directives to prevent naming collisions with user-defined schema types or directives.

```graphql example
extend type Bar
  @abc_variable(name: "def" select: "def" subgraph: "def") {

}

schema
  @fusion(prefix: "abc", version: "2023-12") {

}
```

If the `prefixSelf` argument is set to `true`, the prefix will also be applied to the `@fusion` directive itself.

```graphql example
extend type Bar
  @abc_variable(name: "def" select: "def" subgraph: "def") {

}

schema @abc_fusion(prefix: "abc" prefixSelf: true, version: "2023-12") {

}
```

**Arguments:**

- `version`: Specifies the version of the Fusion specification the document aligns with. The version string consists of the year and month the spec was released, e.g., `2023-12`.
- `prefix`: This string defines the prefix for Fusion-specific types and directives.
- `prefixSelf`: A boolean that, when set to `true`, applies the specified prefix to the `@fusion` directive itself.

### Name

```graphql
scalar Name
```

The scalar `Name` represents a valid GraphQL type name.

### Selection

```graphql
scalar Selection
```

The scalar `Selection` represents a GraphQL field selection syntax.

```graphql example
abc(def: 1) { ghi }
```

## Subgraph Configuration

Composition directives offer instructions for the schema composition process, detailing type system member semantics and specifying type transformations.

### @is

```graphql
directive @is(
  field: Selection
  coordinate: SchemaCoordinate
) on FIELD_DEFINITION | ARGUMENT_DEFINITION | INPUT_FIELD_DEFINITION
```

The `@is` directive is utilized to establish semantic equivalence between disparate type system members across distinct subgraphs, which the schema composition uses to connect types.

In the following example, the directive is used to signal that the `id` argument and the field `id` on the return type `Person` are semantically the same. This information is used to infer an entity resolver for `Person` from the field.

```graphql example
extend type Query {
  personById(id: ID! @is(field: "id")): Person
}
```

The `@is` directive can also refer to nested fields relative to `Person` and is not limited to a single argument.

```graphql example
extend type Query {
  personByAddressId(id: ID! @is(field: "address { id }")): Person
}
```

The directive can also establish semantic equivalence between two output fields. In this example, the field `productSKU` is semantically equivalent to the field `sku` on `Product`, allowing the schema composition to infer the connection of the `Product` with the `Review` type.

```graphql example
extend type Review {
  productSKU: ID! @is(coordinate: "Product.sku") @private
  product: Product @resolve
}
```

The `@is` directive can use either the `field` or `coordinate` argument. If both are specified, the schema composition must fail.

```graphql counter-example
extend type Review {
  productSKU: ID!
    @is(coordinate: "Product.sku", field: "product { sku }")
    @private
  product: Product @resolve
}
```

**Arguments:**

- `field`: Represents a GraphQL field selection syntax that refers to fields relative to the current type.
- `coordinate`: Represents a schema coordinate that refers to a type system member.

### @require

```graphql
directive @require(
  field: Selection
) on ARGUMENT_DEFINITION | INPUT_FIELD_DEFINITION
```

The `@require` directive is used to express argument value requirements with other subgraphs. Arguments annotated with the `@require` directive are removed from the public exposed schema and will be resolved by variables.

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

This can also be done by using input types. All fields of the input type that match the required output type are required. If the input type is only used to express a requirement it is removed from the public schema.

```graphql example
type Product {
  id: ID!
  delivery(
    zip: String!
    dimension: ProductDimensionInput! @require(field: "dimension"))
  ): DeliveryEstimates
}
```

### @resolve

```graphql
directive @resolve(select: Selection, from: Name) repeatable on FIELD_DEFINITION
```

The `@resolve` directive allows to explicitly define a resolver to fetch data from a subgraph. If the annotated field name and its arguments match a query field on any subgraph, the schema composition will infer the resolver, and it is not necessary to specify the `select` argument.

```graphql example
extend type Product {
  calculateDelivery(zip: String!, width: Float!, height: Float!): Int! @resolve
}
```

This is also true if arguments can be inferred from the current `Product` type.

```graphql example
extend type Product {
  sku: String!
  calculateDelivery(zip: String!, width: Float!, height: Float!): Int! @resolve
}

extend type Query {
  calculateDelivery(
    productSku: String @is(coordinate: "Product.sku")
    zip: String!
    width: Float!
    height: Float!
  ): Int!
}
```

The `select` argument of the `@resolve` directives specifies a GraphQL field selection syntax to resolve the data needed for the annotated field. It must be specified if the annotated field name does not match the target root field.

```graphql example
extend type Product {
  calculateDelivery(zip: String!, width: Float!, height: Float!): Int!
    @resolve(select: "estimateDelivery")
}
```

The field's arguments only have to be specified if they do not match the target field's arguments.

```graphql example
extend type Product {
  calculateDelivery(zip: String!, width: Float!, height: Float!): Int!
    @resolve(select: "estimateShipping(addressFilter: { zip: $zip })")
}

extend type Query {
  estimateShipping(
    addressFilter: AddressFilterInput!
    width: Float!
    height: Float!
  ): Int!
}
```

Arguments that match are automatically mapped even if only one argument was explicitly specified. The constant `$__unspecified` can be used to leave a field unspecified'.

```graphql example
extend type Product {
  calculateDelivery(zip: String!, width: Float!, height: Float!): Int!
    @resolve(
      select: "estimateShipping(addressFilter: { zip: $zip }, width: $__unspecified)"
    )
}
```

The `from` argument defines the subgraph to which the `@resolve` directive shall be bound. If `from` is not explicitly specified, the schema composition will bind possible resolvers from all subgraphs.

```graphql example
extend type Product {
  calculateDelivery(zip: String!, width: Float!, height: Float!): Int!
    @resolve(select: "estimateShipping", from: "Shipping")
}
```

**Arguments:**

- `select`: Represents GraphQL field syntax and binds the annotated field to a GraphQL root field.
- `from`: Pins the current resolve declaration to a specific subgraph.

### @declare

```graphql
directive @declare(
  variable: Name!
  select: Selection!
  from: Name
) repeatable on FIELD_DEFINITION
```

`@declare` allows to specify variables selected from the Fusion graph relative to the current type. The declared variables can be used within a `@resolve` directive.

```graphql example
extend type Product {
  dimension: ProductDimension
    @declare(variable: "productId", select: "id")
    @resolve(select: "productDimensionByProductId(id: $productId)")
}
```

Variables can also be constructed from private fields specific to a subgraph.

```graphql example
extend type Product {
  dimension: ProductDimension
    @declare(variable: "productId", select: "internalId", from: "subgraph-a")
    @resolve(select: "productDimensionByProductId(id: $productId)")
}
```

The `select` argument of the `@declare` directive is like the `select` argument of the `@resolve` directive GraphQL field syntax and can refer to fields relative to the current type, in this case, the `Product` type.

```graphql example
extend type Product {
  dimension: ProductDimension
    @declare(
      variable: "productId"
      select: "internal { Id }"
      from: "subgraph-a"
    )
    @resolve(select: "productDimensionByProductId(id: $productId)")
}
```

**Arguments:**

- `variable`: Defines the variable name that can be used within the `select` argument of the `@resolve` directive.
- `select`: Represents GraphQL field syntax and refers to a field relative to the current type.
- `from`: Pins the current declaration to a specific subgraph.

**Arguments:**

- `field`: Represents GraphQL field syntax and refers to a field relative to the current type that represents the requirement.

### @tag

```graphql
directive @tag(
  name: String
) repeatable on OBJECT | INTERFACE | FIELD_DEFINITION | UNION | ENUM | ENUM_VALUE | INPUT_OBJECT | INPUT_FIELD_DEFINITION | SCALAR | SCHEMA
```

The `@tag` directive is used by the schema composition process to include/exclude type members or files. The composition will still use removed parts to build resolvers and optimize data fetching.

```graphql example
input DogInput @tag(name: "Animal") {
  name: String @tag(name: "Internal")
}
```

**Arguments:**

- `name`: The name of the tag.

### @private

```graphql
directive @private on FIELD_DEFINITION
```

The `@private` directive can be annotated on field definitions and will prevent the schema composition from including it from the public scheam. However, this field can still be used bay the schema composition for building resolvers or variables.

### @rename

```graphql
directive @rename(
  coordinate: SchemaCoordinate!
  newName: Name!
) repeatable on SCHEMA
```

The `@rename` directive allows to rename type system members by pointing to the schema coordinate of that type system member. This avoids repeating the type system member just to rename it.

```graphql example
extend schema
  @rename(coordinate: "Foo.bar", newName: "baz")
  @rename(coordinate: "Email_Address", newName: "EmailAddress") {

  }
```

### @remove

```graphql
directive @remove(coordinate: SchemaCoordinate!) repeatable on SCHEMA
```

The `@remove` directive allows to removing type system members by pointing to the schema coordinate of that type system member. This avoids repeating the type system member just to remove it.

### SchemaCoordinate

```graphql
scalar SchemaCoordinate
```

The `SchemaCoordinate` scalar represents a schema coordinate syntax.

```graphql example
Product.id
```

```graphql example
Product.estimateDelivery(zip:)
```

## Gateway Configuration

### @variable

```graphql
directive @variable(
  name: Name!
  select: Selection
  value: Value
  subgraph: Name!
) repeatable on OBJECT | FIELD_DEFINITION
```

The variable directive specifies how state is mapped for a resolver from subgraphs.

Variables can be declared on object types in order to provide state for entity resolvers.

```graphql example
type User
  @variable(name: "User_id" select: "id" subgraph: "Account") {

}
```

They also can be declared on field definitions in order to provide state for explecit field resolvers.

```graphql example
type User {
  address: Address @variable(name: "User_id", select: "id", subgraph: "Account")
}
```

```graphql example
type User 
  @variable(name: "User_id", select: "id", subgraph: "Account")
  @variable(name: "Entity", value: "{ id: $User_id }", subgraph: "Account") {

}
```

```graphql counter-example
type User {
  address: Address @variable(name: "User_id", select: "id", value: "id" subgraph: "Account")
}
```

### @resolver

```graphql
directive @resolver(
  operation: OperationDefinition!
  kind: ResolverKind
  subgraph: Name!
) repeatable on OBJECT | FIELD_DEFINITION
```

The resolver directives specifies how data for a type or field can be fetched.

The `operation` argument contains the GraphQL operation definition syntax which specifies
the requirements for this resolver as variables.

```graphql example
type User
  @resolver(operation: "query($User_id: Int!) { userById(id: $User_id) }" subgraph: "Account") {

}
```

More complex operation can use fragments. There is always a fragment definition specified by the query plan with the name of the return type.

```graphql example
type User
  @resolver(operation: "query($User_id: ID!) { node(id: $User_id) { ... on User { ... User } } }" subgraph: "Account") {

}
```

This allows the query plan to deal with interface fields as entity resolvers and pull up data from lower level fields.

```graphql example
type Address
  @resolver(operation: "query($User_id: ID!) { node(id: $User_id) { ... on User { address { ... Address } } } }" subgraph: "Account") {

}
```

The `kind` of the resolver specifies how data is fetched.

```graphql example
type Address
  @resolver(operation: "query($User_id: [ID!]!) { nodes(ids: $User_id) { ... on User { ... User } } }" kind: BATCH subgraph: "Account") {

}
```

### @source

```graphql
directive @source(
  subgraph: Name!
  name: Name
) repeatable on OBJECT | FIELD_DEFINITION | ENUM | ENUM_VALUE | INPUT_OBJECT | INPUT_FIELD_DEFINITION | SCALAR
```

The `@source` directive specifies on which subgraph a type system member is available and what it's name is on that subgraph.

### @nodes

```graphql
directive @node(types: [Name!]!, subgraph: Name!) repeatable on SCHEMA
```

When a Gateway implements the GraphQL Global Object Identification Specification the `@node` directive specifies from which subgraph the gateway is allowed to resolve the node. It is allowed for node types to be resolve from multiple subgraphs.

```graphql example
schema
  @node(types: [ "User" ], subgraph: "Accounts")
  @node(types: [ "ProducReview", "User" ], subgraph: "Reviews") {
}
```

### @transport

```graphql
directive @transport(
  subgraph: Name!
  kind: String!
  location: URI!
  group: String
) repeatable on SCHEMA
```

### ResolverKind

```graphql
enum ResolverKind {
  FETCH
  BATCH
  SUBSCRIBE
}
```

### Value

The `Value` scalar represent GraphQL value syntax.

```graphql
scalar Value
```