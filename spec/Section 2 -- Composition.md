# Composition

The GraphQL Fusion composition describes the process ...

## Directives

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
  productSKU: ID! @is(coordinate: "Product.sku") @internal
  product: Product @resolve
}
```

The `@is` directive can use either the `field` or `coordinate` argument. If both are specified, the schema composition must fail.

```graphql counter-example
extend type Review {
  productSKU: ID!
    @is(coordinate: "Product.sku", field: "product { sku }")
    @internal
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

Variables can also be constructed from internal fields specific to a subgraph.

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

### @tag

```graphql
directive @tag(
  name: String
) repeatable on OBJECT | INTERFACE | FIELD_DEFINITION | UNION | ENUM | ENUM_VALUE | INPUT_OBJECT | INPUT_FIELD_DEFINITION | SCALAR
```

The `@tag` directive is used by the schema composition process to include/exclude type members or files. The composition will still use removed parts to build resolvers and optimize data fetching.

```graphql example
input DogInput @tag(name: "Animal") {
  name: String @tag(name: "Internal")
}
```

**Arguments:**

- `name`: The name of the tag.

### @internal

```graphql
directive @internal on OBJECT | INTERFACE | FIELD_DEFINITION | UNION | ENUM | ENUM_VALUE | INPUT_OBJECT | INPUT_FIELD_DEFINITION | SCALAR
```

The `@internal` directive can be annotated on field definitions and will prevent the schema composition from including it from the public scheam. However, this field can still be used by the schema composition for building resolvers or variables.

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

## Prune

Before the validation the subgraph schemas will be pruned, things that need to be removed will be removed and things that need to be renamed will be renamed.

- Remove
- Rename

## Validate

A GraphQL Fusion composition tool ensures that the provided subgraph schemas are unambiguous and mistake-free in the context of the composed Fusion graph. The aim here is to guarantee that a composed Fusion graph will have no query errors due to inconsistent schemas.

An request against a inconsistent Fusion graph is still technically executable, and will always produce a stable result as defined by the algorithms in the Execution section, however that result may be ambiguous, surprising, or unexpected relative to a request containing validation errors, so a Fusion composition tool must ensure that subgraphs involved in the Fusion graph from allow for consistent query planning during execution.

Typically validation is performed in the context of the composition.

### Types

#### Semantical Equvalence

**Error Code**

F0001

**Formal Specification**

- Let {typesByName} be the set of all types accross all subgraphs involved in the schema composition by their given type name.
- Given each pair of types {typeA} and {typeB} in {typesByName}
  - {typeA} and {typeB} must have the same kind

**Explanatory Text**

GraphQL Fusions considers types with the same name accross subgraphs as semantically equivalent and mergable.

Types that do not have the same kind are considered not mergeable.

```graphql example
type User {
  id: ID!
  name: String!
  displayName: String!
  birthdate: String!
}

type User {
  id: ID!
  name: String!
  reviews: [Review!]
}
```

```graphql example
scalar DateTime

scalar DateTime
```

```graphql counter-example
type User {
  id: ID!
  name: String!
  displayName: String!
  birthdate: String!
}

scalar User
```

```graphql counter-example
enum UserKind {
  A
  B
  C
}

scalar UserKind
```

### Composite Types

#### Field Types Mergable

**Error Code**

F0002

**Formal Specification**

- Let {fieldsByName} be a map of field lists where the key is the name of a field and the value is a list of fields from mergable types from different subgraphs with the same name.
- for each {fields} in {fieldsByName}
  - {FieldsAreMergable(fields)} must be true.

FieldsAreMergable(fields):

- Given each pair of members {fieldA} and {fieldB} in {fields}:
  - Let {typeA} be the type of {fieldA}
  - Let {typeB} be the type of {fieldB}
  - {SameTypeShape(typeA, typeB)} must be true.

SameTypeShape(typeA, typeB):

- If {typeA} is Non-Null:
  - If {typeB} is nullable
    - Let {innerType} be the inner type of {typeA}
    - return SameTypeShape({innerType}, {typeB})
- If {typeB} is Non-Null:
  - If {typeA} is nullable
    - Let {innerType} be the inner type of {typeB}
    - return {SameTypeShape(typeA, innerType)}
- If {typeA} or {typeB} is List:
  - If {typeA} or {typeB} is not List, return false.
  - Let {innerTypeA} be the item type of {typeA}.
  - Let {innerTypeB} be the item type of {typeB}.
  - return {SameTypeShape(innerTypeA, innerTypeB)}
- If {typeA} and {typeB} are not of the same kind
  - return false
- If {typeA} and {typeB} do not have the same name
  - return false

**Explanatory Text**

Fields on mergable objects or interfaces with that have the same name are considered semantically equivalent and mergable when they have a mergable field type.

Fields with the same type are mergable.

```graphql example
type User {
  birthdate: String
}

type User {
  birthdate: String
}
```

Fields with different nullability are mergable, resulting in merged field with a nullable type.

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

Fields are not mergable if the named types are different in kind or name.

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

#### Argument Types Mergable

**Error Code**

F0004

**Formal Specification**

- Let {fieldsByName} be a map of field lists where the key is the name of a field and the value is a list of fields from mergable types from different subgraphs with the same name.
- for each {fields} in {fieldsByName}
  - if {FieldsInSetCanMerge(fields)} must be true.

FieldsAreMergable(fields):

- Given each pair of members {fieldA} and {fieldB} in {fields}:
  - {ArgumentsAreMergable(fieldA, fieldB)} must be true.

ArgumentsAreMergable(fieldA, fieldB):

- Given each pair of arguments {argumentA} and {argumentB} in {fieldA} and {fieldB}:
  - Let {typeA} be the type of {argumentA}
  - Let {typeB} be the type of {argumentB}
  - {SameTypeShape(typeA, typeB)} must be true.

**Explanatory Text**

Fields on mergable objects or interfaces with that have the same name are considered semantically equivalent and mergable when they have a mergable argument types.

Fields when all matching arguments have a mergable type.

```graphql example
type User {
  field(argument: String): String
}

type User {
  field(argument: String): String
}
```

Arguments that differ on nullability of an argument type are mergable.

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

Arguments are not mergable if the named types are different in kind or name.

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

#### Arguments Mergable

**Error Code**

FXXXX

**Formal Specification**


**Explanatory Text**


```graphql example
type User {
  field(a: String): String
}

type User {
  field(a: String): String
}
```

```graphql counter-example
type User {
  field(a: String): String
}

type User {
  field(b: String): String
}
```

#### Required Arguments Cannot Be Internal

**Error Code**

F0014

**Formal Specification**

- Let {subgraphs} be a list of all subgraphs.
- For each {subgraph} in {subgraphs}:
  - Let {arguments} be the set of all arguments in {subgraph}.
  - For each {argument} in {arguments}:
    - If {IsExposed(argument)} is false
      - Let {type} be the type of {argument}
      - {type} must not be nullable

**Explanatory Text**

Required arguments of fields or directives (i.e., arguments that are non-nullable) must be exposed in the composed schema. Removing a required argument from the composed schema creates a contradiction where an argument is necessary for the operation in the composed schema but is not available for external use.

The following example illustrates a valid scenario where a required argument is not marked as `@internal`:

```graphql example
type Query {
  field1(arg1: Int!): [Item]
}

type Query {
  field1(arg1: Int!): [Item]
}
```

In this counter example, the `arg1` argument is essential for field `field1` in one of the subgraph and is marked as required, but it is also marked as `@internal`, violating the rule that required arguments cannot be internal:

```graphql counter-example
type Query {
  field1(arg1: Int!): [Item]
}

type Query {
  field1(arg1: Int! @internal): [Item]
}
```

In this counter example, the `arg1` argument is essential for field `field1` in one of the subgraph but does not exist in the other subgraph, violating the rule that required arguments cannot be internal:

```graphql counter-example
type Query {
  field1(arg1: Int!): [Item]
}

type Query {
  field1: [Item]
}
```

The same rule applied to directives. The following counter-example illustrate scenario where a directive argument is essential for a directive in one subgraph, but does not exist in the other subgraph:

```graphql counter-example
directive @directive1(arg1: Int!) on FIELD

directive @directive1 on FIELD
```

#### Public Fields Cannot Reference Internals

**Error Code**

F0016

**Formal Specification**

- Let {subgraphs} be a list of all subgraphs.
- For each {subgraph} in {subgraphs}:
  - Let {fields} be the set of all fields of the composite types in {subgraph}
  - For each {field} in {fields}:
    - If {IsExposed(field)} is true
      - Let {namedType} be the named type that {field} references
      - {IsExposed(namedType)} must be true

**Explanatory Text**

In a composed schema, a field within a composite type must only reference types that are exposed. This requirement guarantees that public types do not reference internal structures which are intended for internal use.

Here is a valid example where a public field references another public type:

```graphql example
type Object1 {
  field1: String!
  field2: Object2
}

type Object2 {
  field3: String
}

type Object1 {
  field1: String!
  field4: Int
}
```

This is another valid case where the internal type is not exposed in the composed schema:

```graphql example
type Object1 {
  field1: String!
  field2: Object2 @internal
}

type Object2 @internal {
  field3: String
}

type Object1 {
  field1: String!
  field4: Int
}
```

However, it becomes invalid if a public field in one subgraph references a type that is marked as `@internal` in another subgraph:

```graphql counter-example
type Object1 {
  field1: String!
  field2: Object2
}

type Object2 {
  field3: String
}

type Object1 {
  field1: String!
  field2: Object2 @internal
}

type Object2 @internal {
  field3: String
  field5: Int
}
```
### Object Types

#### Empty Merged Object Type

**Error Code**

F0019

**Formal Specification**

- Let {types} be the set of all object types across all subgraphs
- For each {type} in {types}:
  - {IsObjectTypeEmpty(type)} must be false.

IsObjectTypeEmpty(type):

- Let {fields} be a set of all fields of all types with coordinate and kind {type} across all subgraphs
- For each {field} in {fields}:
  - If {IsExposed(field)} is true
    - return false
- return true

**Explanatory Text**

For object types defined across multiple subgraphs, the merged object type is the superset of all fields defined in these subgraphs. However, any field marked with `@internal` in any subgraph is hidden and not included in the merged object type. An object type with no fields, after considering `@internal` markings, is considered empty and invalid.

In the following example, the merged object type `ObjectType1` is valid. It includes all fields from both subgraphs, with `field2` being hidden due to the `@internal` directive in one of the subgraphs:

```graphql
type ObjectType1 {
  field1: String 
  field2: Int @internal
}

type ObjectType1 {
  field1: String
  field2: Int
  field3: Boolean
}
```

This counter-example demonstrates an invalid merged object type. In this case, `ObjectType1` is defined in two subgraphs, but all fields are marked as `@internal` in at least one of the subgraphs, resulting in an empty merged object type:

```graphql counter-example
type ObjectType1 {
  field1: String @internal
  field2: Boolean 
}

type ObjectType1 {
  field1: String 
  field2: Boolean @internal
}
```

### Input Object Types

#### Input Field Types Mergable

**Error Code**

F0005

**Formal Specification**

- Let {fieldsByName} be a map of field lists where the key is the name of a field and the value is a list of fields from mergable input types from different subgraphs with the same name. 
- For each {fields} in {fieldsByName}:
  - if {InputFieldsAreMergable(fields)} must be true.

InputFieldsAreMergable(fields):

- Given each pair of members {fieldA} and {fieldB} in {fields}:
  - Let {typeA} be the type of {fieldA}.
  - Let {typeB} be the type of {fieldB}.
  - {SameTypeShape(typeA, typeB)} must be true.

**Explanatory Text**

The input fields of input objects with the same name must be mergable. This rule ensures that input objects with the same name in different subgraphs have fields that can be consistently merged without conflict.

Input fields are considered mergable when they share the same name and have compatible types. The compatibility of types is determined by their structure (lists), excluding nullability. Mergable input fields with different nullability are considered mergable, and the resulting merged field will be the most permissive of the two.

In this example, the field `field` in `Input1` has compatible types across subgraphs, making them mergable:

```graphql example
input Input1 {
  field: String!
}

input Input1 {
  field: String
}
```

Here, the field `tags` in `Input1` is a list type with compatible inner types, satisfying the mergable criteria:

```graphql example
input Input1 {
  tags: [String!]
}

input Input1 {
  tags: [String]!
}

input Input1 {
  tags: [String]
}
```

In this counter-example, the field `field` in `Input1` has incompatible types (`String` and `DateTime`), making them not mergable:

```graphql counter-example
input Input1 {
  field: String!
}

input Input1 {
  field: DateTime!
}
```

Here, the field `tags` in `Input1` is a list type with incompatible inner types (`String` and `DateTime`), violating the mergable rule:

```graphql counter-example
input Input1 {
  tags: [String]
}

input Input1 {
  tags: [DateTime]
}
```

#### Input With Different Fields

**Error Code**

F0006

**Formal Specification**

- Let {inputsByName} be a map where the key is the name of an input object type, and the value is a list of all input object types from different subgraphs with that name.
- For each {listOfInputs} in {inputsByName}:
  - {InputFieldsAreMergable(listOfInputs)} must be true.

InputFieldsAreMergable(inputs):

- Let {fields} be the set of all field names of the first input object in {inputs}.
- For each {input} in {inputs}:
  - Let {inputFields} be the set of all field names of {input}.
  - {fields} must be equal to {inputFields}.

**Explanatory Text**

This rule ensures that input object types with the same name across different subgraphs have identical sets of field names. Consistency in input object fields across subgraphs is required to avoid conflicts and ambiguities in the composed schema. This rule only checks that the field names are the same, not that the field types are the same. Field types are checked by the [Input Field Types Mergable](#sec-Input-Field-Types-Mergable) rule.

When an input object is defined with differing fields across subgraphs, it can lead to issues in query execution. A field expected in one subgraph might be absent in another, leading to undefined behavior. This rule prevents such inconsistencies by enforcing that all instances of the same named input object across subgraphs have a matching set of field names.

In this example, both subgraphs define `Input1` with the same field `field1`, satisfying the rule:

```graphql example
input Input1 {
  field1: String
}

input Input1 {
  field1: String
}
```

Here, the two definitions of `Input1` have different fields (`field1` and `field2`), violating the rule:

```graphql counter-example
input Input1 {
  field1: String
}

input Input1 {
  field2: String
}
```

#### Empty Merged Input Object Type

**Error Code**

F0010

**Formal Specification**

- Let {inputs} be the set of all input object types across all subgraphs
- For each {input} in {inputs}:
  - If {IsExposed(input)} is true
    - {IsInputObjectTypeEmpty(input)} must be false
  

IsInputObjectTypeEmpty(input):

- Let {fields} be a set of all input fields across all subraphs with coordinate {input} 
- For each {field} in {fields}:
  - If {IsExposed(field)} is true
    - return false

**Explanatory Text**

When an input object type is defined in multiple subgraphs, only common fields and fields not flagged as `@internal` are included in the merged input object type. If this process results in an input object type with no fields, the input object type is considered empty and invalid.

In the following example, the merged input object type `Input1` is valid. The type is defined in two subgraphs:

```graphql
input Input1 {
  field1: String @internal
  field2: Int
}

input Input1 {
  field1: String
  field2: Int
}
```

In the following example, the merged input object type `Input1` is valid and contains `field1` as it exists in both subgraphs. The type is defined in two subgraphs:

```graphql
input Input1 {
  field1: String 
}

input Input1 {
  field1: String
  field2: Int
}
```

In the following example, the merged input object type `Input1` is invalid. The type is defined in two subgraphs, but do not have any common fields.
```graphql counter-example
input Input1 {
  field1: String 
}

input Input1 {
  field2: String
}
```

This counter-example shows an invalid merged input object type. The `Input1` type is defined in two subgraphs, but both `field1` and `field2` are flagged as `@internal` in one of the subgraphs:

```graphql counter-example
input Input1 {
  field1: String @internal
  field2: Int @internal
}

input Input1 {
  field1: String
  field2: Int
}
```

In this counter-example, the `Input1` type is defined in two subgraphs, but `field1` is flagged as `@internal` in one subgraph and `field2` is flagged as `@internal` in the other subgraph, resulting in an empty merged input object type:

```graphql counter-example
input Input1 {
  field1: String @internal
  field2: Int
}

input Input1 {
  field1: String
  field2: Int @internal
}
```

In the following example, the merged input object type `Input1` is invalid. The type is defined in two subgraphs, but do not have the common field `field1` is flagged as `@internal` in one of the subgraphs:

```graphql counter-example
input Input1 {
  field1: String  @internal
}

input Input1 {
  field1: String
}
```


#### Input Field Default Mismatch

**Error Code**

F0011

**Formal Specification**

- Let {inputFieldsByName} be a map where the key is the name of an input field and the value is a list of input fields from different subgraphs from the same type with the same name.
- For each {inputFields} in {inputFieldsByName}:
  - Let {defaultValues} be a set containing the default values of each input field in {inputFields}.
  - If the size of {defaultValues} is greater than 
    - {InputFieldsHaveConsistentDefaults(inputFields)} must be false.

InputFieldsHaveConsistentDefaults(inputFields):

- Given each pair of input fields {inputFieldA} and {inputFieldB} in {inputFields}:
  - If the default value of {inputFieldA} is not equal to the default value of {inputFieldB}:
    - return false
- return true

**Explanatory Text**

Input fields in different subgraphs that have the same name are required to have consistent default values. This ensures that there is no ambiguity or inconsistency when merging schemas from different subgraphs.

A mismatch in default values for input fields with the same name across different subgraphs will result in a schema composition error.

In the the following example both subgraphs have an input field `field1` with the same default value. This is valid:

```graphql example
input Filter {
  field1: Enum1 = Value1
}

enum Enum1 {
  Value1
  Value2
}

input Filter {
  field1: Enum1 = Value1
}

enum Enum1 {
  Value1
  Value2
}
```

In the following example both subgraphs define an input field `field1` with different default values. This is invalid:

```graphql counter-example
input Filter {
  field1: Int = 10
}

input Filter {
  field1: Int = 20
}
```

#### Public Input Fields Cannot Reference Internals

**Error Code**

F0015

**Formal Specification**

- Let {subgraphs} be a list of all subgraphs.
- For each {subgraph} in {subgraphs}:
  - Let {fields} be the set of all fields of the input types in {subgraph}
  - For each {field} in {fields}:
    - If {IsExposed(field)} is true
      - Let {namedType} be the named type that {field} references
      - {IsExposed(namedType)} must be true

**Explanatory Text**

In a composed schema, a field within a input type must only reference types that are exposed. This requirement guarantees that public types do not reference internal structures which are intended for internal use.

A valid case where a public input field references another public input type:

```graphql example
input Input1 {
  field1: String!
  field2: Input2
}

input Input2 {
  field3: String
}

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
  field2: Input2
}

input Input2 {
  field3: String
}

input Input1 {
  field1: String!
  field4: Int
}
```

An invalid case where a public input field in one subgraph references an input type that is marked as `@internal` in another subgraph:

```graphql counter-example
input Input1 {
  field1: String!
  field2: Input2
}

input Input2 {
  field3: String
}

input Input1 {
  field1: String!
  field4: Int
}

input Input2 @internal {
  field3: String
}
```

#### Required Input Fields cannot be internal

**Error Code**

F0012

**Formal Specification**

- Let {subgraphs} be a list of all subgraphs.
- For each {subgraph} in {subgraphs}:
  - Let {inputs} be the set of all input object types in the {subgraph} 
  - For each {input} in {inputs}:
    - Let {fields} be a list of fields of {input} 
    - For each {field} in {fields}:
      - If {IsExposed(field)} is false
        - Let {type} be the type of {field}
        - {type} must not be nullable

**Explanatory Text**

Input object types can be defined in multiple subgraphs. Required fields in these input object types (i.e., fields that are non-nullable) must be exposed in the composed graph. 
A required internal field would create a contradiction where a field is both necessary for the operation in the composed schema but also not available for external use.

This example shows a valid scenario where the required field is not marked as `@internal`:

```graphql example
input InputType1 {
  field1: String!
}

input InputType1 {
  field1: String!
}
```

Consider the following example where an input object type `InputType1` includes a required field `field1`:

```graphql counter-example
input InputType1 {
  field1: String!
}

input InputType1 {
  field1: String! @internal
}
```

In this counter-example, the field `field1` is only defined in one subgraph, so will not be included in the composed schema. Therefor this field cannot be non-nullable:

```graphql counter-example
input InputType1 {
  field1: String!
  field2: String!
}

input InputType1 {
  field2: String!
}
```

### Enum Types

#### Values Must Be The Same Across Subgraphs

**Error Code**

F0003

**Formal Specification**

- Let {enumsByName} be a map where the key is the name of an enum type, and the value is a list of all enum types from different subgraphs with that name.
- For each {listOfEnum} in {enumsByName}:
  - {EnumAreMergable(listOfEnum)} must be true.

EnumAreMergable(enums):

- Let {values} be the set of all values of the first enum in {enums}
- For each {enum} in {enums}
  - Let {enumValues} be the set of all values of {enum}
  - {values} must be equal to {enumValues}

**Explanatory Text**

This rule ensures that enum types with the same name across different subgraphs in a GraphQL Fusion schema have identical sets of values. Enums, must be consistent across subgraphs to avoid conflicts and ambiguities in the composed schema.

When an enum is defined with differing values across subgraphs, it can lead to confusion and errors in query execution. For instance, a value valid in one subgraph might be passed to another where it's unrecognized, leading to unexpected behavior or failures. This rule prevents such inconsistencies by enforcing that all instances of the same named enum across subgraphs have an exact match in their values.

In this example, both subgraphs define `Enum1` with the same value `BAR`, satisfying the rule:

```graphql example
enum Enum1 {
  BAR
}

enum Enum1 {
  BAR
}
```

Here, the two definitions of `Enum1` have different values (`BAR` and `Baz`), violating the rule:

```graphql counter-example
enum Enum1 {
  BAR
}

enum Enum1 {
  Baz
}
```

#### Default Value Uses Internals

**Error Code**

F0008

**Formal Specification**

- {ValidateArgumentDefaultValues()} must be true.
- {ValidateInputFieldDefaultValues()} must be true.

ValidateArgumentDefaultValues():

- Let {arguments} be all arguments of fields and directives across all subgraphs 
- For each {argument} in {arguments}
  - If {IsExposed(argument)} is true and has a default value:
    - Let {defaultValue} be the default value of {argument}
    - If not ValidateDefaultValue(defaultValue)
      - return false
- return true

ValidateInputFieldDefaultValues():

- Let {inputFields} be all input fields of across all subgraphs
- For each {inputField} in {inputFields}:
  - Let {type} be the type of {inputField}
  - If {IsExposed(inputField)} is true and {inputField} has a default value:
      - Let {defaultValue} be the default value of {inputField}
      - if ValidateDefaultValue(defaultValue) is false
        - return false
- return true

ValidateDefaultValue(defaultValue):

- If {defaultValue} is a ListValue:
  - For each {valueNode} in {defaultValue}:
    - If {ValidateDefaultValue(valueNode)} is false
      - return false
- If {defaultValue} is an ObjectValue:
  - Let {objectFields} be a list of all fields of {defaultValue}
  - Let {fields} be a list of all fields {objectFields} are refering to
  - For each {field} in {fields}:
    - If {IsExposed(field)} is false
      - return false
  - For each {objectField} in {objectFields}:
    - Let {value} be the value of {objectField}
    - return {ValidateDefaultValue(value)}
- If {defaultValue} is an EnumValue:
  - If {IsExposed(defaultValue)} is false
    - return false
- return true

**Explanatory Text**

A default value for an argument in a field must only reference enum values or a input fields that are exposed in the composed schema.
This rule ensures that internal members are not exposed in the composed schema through default values.

In this example the `FOO` value in the `Enum1` enum is not marked with @internal, hence it doesn't violate the rule.

```graphql 
type Query {
  field(type: Enum1 = FOO): [Baz!]!
}

enum Enum1 {
  FOO
  BAR
}
```

This example is a violation of the rule because the default value for the field `field` in type `Input1` references an enum value (`FOO`) that is marked as `@internal`.

```graphql counter-example
type Query {
  field(type: Input1): [Baz!]!
}

input Input1 {
  field: Enum1 = FOO
}

enum Enum1 {
  FOO @internal
  BAR
}
```

```graphql counter-example
type Query {
  field(type: Input1 = {field2: "ERROR"}): [Baz!]!
}

input Input1 {
  field1: String
  field2: String @internal
}
```


This example is a violation of the rule because the default value for the `type` argument in the `field` field references an enum value (`BAR`) that is marked as `@internal`.

```graphql example counter-example
type Query {
  field(type: Enum1 = BAR): [Baz!]!
}

enum Enum1 {
  FOO
  BAR @internal
}
```

#### Empty Merged Enum Type

**Error Code**

F0009

**Formal Specification**

- Let {enumsByName} be the set of all enums across all subgraphs involved in the schema composition grouped by their name.
- For each {enum} in {enumsByName}:
  - If {IsExposed(enum)} is true
    - {IsEnumEmpty(enum)} must be false

IsEnumEmpty(enum):

- Let {values} be a set of enum values for {enum} across all subgraphs
- For each {value} in {values}
  - If {IsExposed(value)} is true
    - return false
- return true
  

**Explanatory Text**

If an enum type is defined in multiple subgraphs, only values not flagged as `@internal` in all subgraphs are included in the merged enum type. If this process results in an enum type with no values, the enum type is considered empty and invalid.

The following example shows a valid merged enum type. The `Enum1` type is defined in two subgraphs:

```graphql
enum Enum1 {
  Value1 @internal
  Value2 
}

enum Enum1 {
  Value1 
  Value2 
}
```

This counter-example shows an invalid merged enum type. The `Enum1` type is defined in two subgraphs, but the `Value1` and `Value2` values are flagged as `@internal` in one of the subgraphs:

```graphql counter-example
enum Enum1 {
  Value1 @internal
  Value2 @internal
}

enum Enum1 {
  Value1 
  Value2 
}
```

In this counter-example, the `Enum1` type is defined in two subgraphs, but the `Value1` value is flagged as `@internal` in one of the subgraphs and the `Value2` value is flagged as `@internal` in the other subgraph which results in an empty merged enum type:

```graphql counter-example
enum Enum1 {
  Value1 @internal
  Value2 
}

enum Enum1 {
  Value1 
  Value2 @internal
}
```
### Interface 

#### Empty Merged Interface Type

**Error Code**

F0018

**Formal Specification**

- Let {interfaces} be the set of all interface types across all subgraphs.
- For each {interface} in {interfaces}:
  - If {IsExposed(interface)} is true
    - {IsInterfaceTypeEmpty(interface)} must be false.
  
IsInterfaceTypeEmpty(interface):

- Let {fields} be a set of all fields across all subgraphs with coordinate and kind of {interface}
- For each {field} in {fields}:
  - If {IsExposed(field)} is true
    - return false
- return true

**Explanatory Text**

When an interface type is defined in multiple subgraphs, only common fields and fields not flagged as `@internal` are included in the merged interface type. If this process results in an interface type with no fields, the interface type is considered empty and invalid.

In the following example, the merged interface type `Interface1` is valid. The type is defined in two subgraphs:

```graphql example
interface Interface1 {
  field1: String @internal
  field2: Int
}

interface Interface1 {
  field1: String
  field2: Int
}
```

In the following example, the merged interface type `Interface1` is valid and contains `field1` as it exists in both subgraphs. The type is defined in two subgraphs:

```graphql example
interface Interface1 {
  field1: String 
}

interface Interface1 {
  field1: String
  field2: Int
}
```

In the following example, the merged interface type `Interface1` is invalid. The type is defined in two subgraphs, but do not have any common fields:

```graphql counter-example
interface Interface1 {
  field1: String 
}

interface Interface1 {
  field2: String
}
```

This counter-example shows an invalid merged interface type. The `Interface1` type is defined in two subgraphs, but both `field1` and `field2` are flagged as `@internal` in one of the subgraphs:

```graphql counter-example
interface Interface1 {
  field1: String @internal
  field2: Int @internal
}

interface Interface1 {
  field1: String
  field2: Int
}
```

In this counter-example, the `Interface1` type is defined in two subgraphs, but `field1` is flagged as `@internal` in one subgraph and `field2` is flagged as `@internal` in the other subgraph, resulting in an empty merged interface type:

```graphql counter-example
interface Interface1 {
  field1: String @internal
  field2: Int
}

interface Interface1 {
  field1: String
  field2: Int @internal
}
```

In the following example, the merged interface type `Interface1` is invalid. The type is defined in two subgraphs, but the common field `field1` is flagged as `@internal` in one of the subgraphs:

```graphql counter-example
interface Interface1 {
  field1: String @internal
}

interface Interface1 {
  field1: String
}
```


### Union 

#### Union Type Members Cannot Reference Internals

**Error Code**

F0017

**Formal Specification**

- Let {subgraphs} be a list of all subgraphs.
- For each {subgraph} in {subgraphs}:
  - Let {unionTypes} be the set of all union types in {subgraph}.
  - For each {unionType} in {unionTypes}:
    - Let {members} be the set of member types of {unionType} 
    - For each {member} in {members}:
      - Let {type} be the type of {member}
      - {IsExposed(type)} must be true

**Explanatory Text**

A union type must not include members where the type is marked as `@internal` in any subgraph. This rule ensures that the composed schema does not expose internal types.

Here is a valid example where all member types of a union are public:

```graphql example
type Object1 {
  field1: String
}

type Object2 {
  field2: Int
}

union Union1 = Object1 | Object2

type Object3 {
  field3: Boolean
}

union Union1 = Object3
```

In this counter-example, the member type `Object2` is marked as `@internal` in the second subgraph, so the composed schema is invalid:

```graphql counter-example
union Union1 = Object1 | Object2

type Object1 {
  field1: String
}

type Object2 {
  field2: Int
}

union Union1 = Object1 | Object2 

type Object1 {
  field1: String
}

type Object2 @internal {
  field2: Int
  field4: Float
}
```

### Schema

#### No Queries

**Error Code**

F0007

**Formal Specification**

- Let {schemas} be the set of all subgraph schemas involved in the composition
- {HasQueryRootType(schemas)} must be true

HasQueryRootType(schemas):

- for each {schema} in {schemas}
  - let {queryRootType} be the query root type of {schema}
  - let {fields} be the fields of {queryRootType}
  - if {fields} is not empty
    - return true
- return false

**Explanatory Text**

A graphQL fusion schema composed from multiple subgraphs must have a query root type to be valid. The query root type is essential. This rule ensures that at least one subgraph in the composition defines a query root type that defines at least one query field.

The examples provided in the test cases demonstrate how the absence or presence of a valid query root type impacts the composition result. A composition that does not meet this requirement should result in an error with the code F0007, indicating the need for at least one query field across the included subgraphs.

```graphql
type Query {
  field1: String
}

type Query {
  field2: String @remove
}
```

In the example, both `field1` and `field2` are marked for removal, leaving no valid query fields:

```graphql counter-example
type Query {
  field1: String @remove
}

type Query @remove {
  field2: String 
}
```

In this case, the presence of `@internal` on `field`1 removes the field from the composed schema, leaving no accessible query fields.

```graphql counter-example
type Query {
  field1: String @internal
}

type Query {
  field1: String 
}
```

### Shared Functions

SameTypeShape(typeA, typeB):

- If {typeA} is Non-Null:
  - If {typeB} is nullable
    - Let {innerType} be the inner type of {typeA}.
    - return {SameTypeShape(innerType, typeB)}.
- If {typeB} is Non-Null:
  - If {typeA} is nullable
    - Let {innerType} be the inner type of {typeB}.
    - return {SameTypeShape(typeA, innerType)}.
- If {typeA} or {typeB} is List:
  - If {typeA} or {typeB} is not List
    - return false.
  - Let {innerTypeA} be the item type of {typeA}.
  - Let {innerTypeB} be the item type of {typeB}.
  - return {SameTypeShape(innerTypeA, innerTypeB)}.
- If {typeA} and {typeB} are not of the same kind
  - return false.
- If {typeA} and {typeB} do not have the same name
  - return false.


IsExposed(member):


- Let {members} be a list of all members across all subgraphs with the same coordinate and kind as {member} 
- If any {members} is marked with `@internal`
  - return false
- If {member} is InputField
  - Let {type} be any input object type that {member} is declared on
  - if {IsExposed(type)} is false
    - return false
  - Let {types} be the list of all types across all subgraphs with the same coordinate and kind as {type}
  - If {member} is in {CommonFields(types)} 
    - return true
  - return false
- If {member} is EnumValue
  - Let {enum} be any enum that {member} is declared on
  - return {IsExposed(enum)}
- If {member} is ObjectField
  - Let {type} be the any type that {member} is declared on
  - return {IsExposed(type)} 
- If {member} is InterfaceField
  - Let {type} be the any type that {member} is declared on
  - If {IsExposed(type)} is false
    - return false
  - Let {types} be the list of all types across all subgraphs with the same coordinate and kind as {type}
  - If {member} is in {CommonFields(types)} 
    - return true
  - return false
- If {member} is Argument
  - If {member} is declared on a field:
    - Let {declaringField} be any field that {member} is declared on
    - If {IsExposed(declaringField)} is false
      - return false
    - Let {declaringFields} be the list of all fields across all subgraphs with the same coordinate and kind as {declaringField}
    - If {member} is in {CommonArguments(declaringFields)} 
      - return true
    - return false
  - If {member} is declared on a directive
    - Let {declaringDirective} be any directive that {member} is declared on
    - If {IsExposed(declaringDirective)} is false
      - return false
    - Let {declaringDirectives} be the list of any directives across all subgraphs with the same coordiante and kind as {declaringDirective} 
    - If {member} is in {CommonArguments(declaringDirectives)} 
      - return true
    - return false
- return true

CommonArguments(members):

- Let {commonArguments} be the set of all arguments of all {members} across all subgraphs
- For each {member} in {members}
  - Let {arguments} be the set of all arguments of {member}
  - Let {commonArguments} be the intersection of {commonArguments} and {arguments}
- return {commonArguments}
    
CommonFields(members):

- Let {commonFields} be the set of all fields of all {members} across all subgraphs
- For each {member} in {members}
  - Let {fields} be the set of all fields of {member}
  - Let {commonFields} be the intersection of {commonFields} and {fields}
- return {commonFields}



## Compose

To compose a gateway configuration the composition tooling must have loaded at least one subgraph configuration.

ComposeConfiguration(subgraphConfigurations):

- For each {subgraphConfiguration} in {subgraphConfigurations}:
- Let {subgraphSchema} be the result of {BuildSubgraphSchema(subgraphConfiguration)}.

BuildSubgraphSchema(subgraphConfiguration):

- Todo

MergeType(typesOrTypeExtensions):

- Todo

MergeSchema(schemaA, schemaB):

- Let {mergedSchema} be an empty SchemaDefinition
- Let {queryTypeA} be the query type of {schemaA}
- Set {mergedSchema} to have query type {queryTypeA}
- Let {mutationTypeA} be the mutation type of {schemaA} or null
- Let {mutationTypeB} be the mutation type of {schemaB} or null
- If {mutationTypeA} is not null
  - Set {mergedSchema} to have mutation type {mutationTypeA}
- If {mutationTypeA} is null and {mutationTypeB} is not null
  - Set {mergedSchema} to have mutation type {mutationTypeB}
- Let {subscriptionTypeA} be the subscription type of {schemaA} or null
- Let {subscriptionTypeB} be the subscription type of {schemaB} or null
- If {subscriptionTypeA} is not null
  - Set {mergedSchema} to have subscription type {subscriptionTypeA}
- If {subscriptionTypeA} is null and {subscriptionTypeB} is not null
  - Set {mergedSchema} to have subscription type {subscriptionTypeB}
- return {mergedSchema}

MergeOutputType(typeA, typeB):

- If {typeA} or {typeB} are `@internal`
  - return null
- Let {outputType} be an empty OutputObjectTypeDefinition
- Let {fields} be a set of field names that are defined on either typeA or typeB
- For each {fieldName} in {fields}:
  - Let {fieldA} be the field with the same name on typeA
  - Let {fieldB} be the field with the same name on typeB
  - If {fieldA} or {fieldB} are `@internal`
    - continue
  - Let {mergedField} be the result of calling MergeOutputField(fieldA, fieldB)
  - Append {mergedField} to {outputType}
- Set {outputType} to have the same name as {typeA}
- Set {outputType} to have {MergeInterfaceImplementation(typeA, typeB)} as its interfaces
- return {outputType}

MergeInputType(typeA, typeB): 

- If {typeA} or {typeB} are `@internal`
  - return null
- Let {inputType} be an empty InputObjectTypeDefinition
- Let {commonFields} be the set of fields names that are defined on both {typeA} and {typeB}
- For each {field} in {commonFields}:
  - Let {fieldA} be the field with the same name on {typeA}
  - Let {fieldB} be the field with the same name on {typeB}
  - If {fieldA} or {fieldB} are `@internal`
    - continue
  - Let {mergedField} be the result of calling MergeInputField(fieldA, fieldB)
  - Append {mergedField} to {inputType}
- Set {inputType} to have the same name as {typeA}
- return {inputType}

MergeInterfaceType(typeA, typeB):

- If {typeA} or {typeB} are `@internal`
  - return null
- Let {interfaceType} be an empty InterfaceTypeDefinition
- Let {fields} be a set of common field names that are defined in {typeA} and {typeB}
- For each {fieldName} in {fields}:
  - Let {fieldA} be the field with the same name on {typeA}
  - Let {fieldB} be the field with the same name on {typeB}
  - If {fieldA} or {fieldB} are `@internal`
    - continue
  - Let {mergedField} be the result of calling {MergeOutputField(fieldA, fieldB)}
  - Append {mergedField} to {interfaceType}
- Set {interfaceType} to have the same name as {typeA}
- Set {interfaceType} to have {MergeInterfaceImplementation(typeA, typeB)} as its interfaces
- return {interfaceType}

MergeUnionType(typeA, typeB):

- If {typeA} or {typeB} are `@internal`
  - return null
- Let {unionType} be an empty UnionTypeDefinition
- Let {types} be a set of type names that are defined on either {typeA} or {typeB}
- Set {unionType} to have the same name as {typeA}
- Set {types} to be the types of {unionType}
- return {unionType}

MergeEnumType(typeA, typeB):

- If {typeA} or {typeB} are `@internal`
  - return null
- Let {enumType} be an empty EnumTypeDefinition
- Let {valuesOfA} be the set of enum values defined on {typeA}
- Set {enumType} to have the same name as {typeA}
- Set {valuesOfA} to be the values of {enumType}
- return {enumType}

MergeInputField(fieldA, fieldB):

- Let {mergedField} be an empty InputFieldDefinition
- Let {typeA} be the type of fieldA
- Let {typeB} be the type of fieldB
- Let {mergedType} be the result of calling {MergeInputFieldType(typeA, typeB)}
- Set {mergedField} to have the same name as {fieldA}
- Set {mergedField} to have {mergedType} as its type
- return {mergedField}

MergeOutputField(fieldA, fieldB):

- Let {mergedField} be an empty OutputFieldDefinition
- Let {typeA} be the type of fieldA
- Let {typeB} be the type of fieldB
- Let {mergedType} be the result of calling {MergeOutputFieldType(typeA, typeB)}
- Set {mergedField} to have the same name as {fieldA}
- Set {mergedField} to have {mergedType} as its type
- Set {mergedField} to have arguments {MergeArguments(fieldA, fieldB)}
- return {mergedField}

MergeArguments(fieldA, fieldB):

- Let {mergedArguments} be an empty list of ArgumentDefinitions
- Let {arguments} be a set of argument names that are defined on either {fieldA} or {fieldB}
- For each {argumentName} in {arguments}:
  - Let {argumentA} be the argument with the same name on {fieldA}
  - Let {argumentB} be the argument with the same name on {fieldB}
  - If {argumentA} or {argumentB} are `@internal`
    - continue
  - Let {mergedArgument} be the result of calling MergeArgument(argumentA, argumentB)
  - Append {mergedArgument} to {mergedArguments}
- return {mergedArguments}

MergeArgument(argumentA, argumentB):

- Let {mergedArgument} be an empty ArgumentDefinition
- Let {typeA} be the type of argumentA
- Let {typeB} be the type of argumentB
- Let {mergedType} be the result of calling {MergeInputFieldType(typeA, typeB)}
- Set {mergedArgument} to have the same name as {argumentA}
- Set {mergedArgument} to have {mergedType} as its type
- return {mergedArgument}

MergeInterfaceImplementation(typeA, typeB):

- Let {interfaceNames} be the set of interface names that are defined on either {typeA} or {typeB}
- Let {validInterfaceNames} be the set of all interface in the composition that are not `@internal`
- Return the intersection of {interfaceNames} and {validInterfaceNames}

MergeInputFieldType(typeA, typeB):

- If {typeA} and {typeB} are the same
  - return {typeA}
- If {typeA} is Non-Null and {typeB} is Non-Null
  - Let {innerTypeA} be the inner type of {typeA}
  - Let {innerTypeB} be the inner type of {typeB}
  - Let {innerType} be {MergeOutputFieldType(innerTypeA, innerTypeB)}
  - return {ToNonNullType(innerType)} 
- If {typeA} is Non-Null and {typeB} is nullable
  - Let {innerTypeA} be the inner type of {typeA}
  - Let {mergedType} be {MergeOutputFieldType(innerTypeA, typeB)}
  - return {ToNonNullType(mergedType)}
- If {typeB} is Non-Null and {typeA} is nullable
  - Let {innerTypeB} be the inner type of {typeB}
  - Let {mergedType} be {MergeOutputFieldType(typeA, innerTypeB)}
  - return {ToNonNullType(mergedType)}
- If both {typeA} and {typeB} are List
  - Let {itemTypeA} be the item type of {typeA}
  - Let {itemTypeB} be the item type of {typeB}
  - Let {innerType} be {MergeOutputFieldType(itemTypeA, itemTypeB)}
  - return {ToListType(innerType)}
- raise composition error

MergeOutputFieldType(typeA, typeB):

- If {typeA} and {typeB} are the same
  - return {typeA}
- If both {typeA} and {typeB} are Non-Null
  - Let {innerTypeA} be the inner type of {typeA}
  - Let {innerTypeB} be the inner type of {typeB}
  - Let {innerType} be {MergeOutputFieldType(innerTypeA, innerTypeB)}
  - return {ToNonNullType(innerType)} 
- If {typeA} is Non-Null and {typeB} is nullable
  - Let {innerTypeA} be the inner type of {typeA}
  - Let {mergedType} be {MergeOutputFieldType(innerTypeA, typeB)}
  - return {ToNullableType(mergedType)}
- If {typeB} is Non-Null and {typeA} is nullable
  - Let {innerTypeB} be the inner type of {typeB}
  - Let {mergedType} be {MergeOutputFieldType(typeA, innerTypeB)}
  - return {ToNullableType(mergedType)}
- If both {typeA} and {typeB} are Lists:
  - Let {itemTypeA} be the item type of {typeA}.
  - Let {itemTypeB} be the item type of {typeB}.
  - Let {innerType} be {MergeOutputFieldType(itemTypeA, itemTypeB)}
  - return {ToListType(innerType)}
- raise composition error

ToNonNullType(type):

- If {type} is Non-Null
  - return {type}
- Let {nonNullType} be a new Non-Null type with {type} as inner type
- return {nonNullType}

ToNullableType(type):

- If {type} is Non-Null
  - Let {innerType} be the inner type of {type}
  - return {innerType}
- return {type}

ToListType(type):

- Let {listType} be a new List type with {type} as item type
- return {listType}

## Finalize
