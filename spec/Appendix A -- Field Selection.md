# Appendix A: Specification of FieldSelectionMap Scalar

## Introduction

This appendix focuses on the specification of the {FieldSelectionMap} scalar
type. {FieldSelectionMap} is designed to express semantic equivalence between
arguments of a field and fields within the result type. Specifically, it allows
defining complex relationships between input arguments and fields in the output
object by encapsulating these relationships within a parsable string format. It
is used in the `@is` and `@require` directives.

To illustrate, consider a simple example from a GraphQL schema:

```graphql
type Query {
  userById(userId: ID! @is(field: "id")): User! @lookup
}
```

In this schema, the `userById` query uses the `@is` directive with
{FieldSelectionMap} to declare that the `userId` argument is semantically
equivalent to the `User.id` field.

An example query might look like this:

```graphql
query {
  userById(userId: "123") {
    id
  }
}
```

Here, it is expected that the `userId` "123" corresponds directly to `User.id`,
resulting in the following response if correctly implemented:

```json
{
  "data": {
    "userById": {
      "id": "123"
    }
  }
}
```

The {FieldSelectionMap} scalar is represented as a string that, when parsed,
produces a {SelectedValue}.

A {SelectedValue} must exactly match the shape of the argument value to be
considered valid. For non-scalar arguments, you must specify each field of the
input type in {SelectedObjectValue}.

```graphql example
extend type Query {
  findUserByName(user: UserInput! @is(field: "{ firstName: firstName }")): User
    @lookup
}
```

```graphql counter-example
extend type Query {
  findUserByName(user: UserInput! @is(field: "firstName")): User @lookup
}
```

### Scope

The {FieldSelectionMap} scalar type is used to establish semantic equivalence
between an argument and fields within a specific output type. This output type
is always a composite type, but the way it's determined can vary depending on
the directive and context in which the {FieldSelectionMap} is used.

For example, when used with the `@is` directive, the {FieldSelectionMap} maps
between the argument and fields in the return type of the field. However, when
used with the `@require` directive, it maps between the argument and fields in
the object type on which the field is defined.

Consider this example:

```graphql
type Product {
  id: ID!
  delivery(
    zip: String!
    size: Int! @require(field: "dimension.size")
    weight: Int! @require(field: "dimension.weight")
  ): DeliveryEstimates
}
```

In this case, `"dimension.size"` and `"dimension.weight"` refer to fields of the
`Product` type, not the `DeliveryEstimates` return type.

Consequently, a {FieldSelectionMap} must be interpreted in the context of a
specific argument, its associated directive, and the relevant output type as
determined by that directive's behavior.

**Examples**

Scalar fields can be mapped directly to arguments.

This example maps the `Product.weight` field to the `weight` argument:

```graphql example
type Product {
  shippingCost(weight: Float @require(field: "weight")): Currency
}
```

This example maps the `Product.shippingWeight` field to the `weight` argument:

```graphql example
type Product {
  shippingCost(weight: Float @require(field: "shippingWeight")): Currency
}
```

Nested fields can be mapped to arguments by specifying the path. This example
maps the nested field `Product.packaging.weight` to the `weight` argument:

```graphql example
type Product {
  shippingCost(weight: Float @require(field: "packaging.weight")): Currency
}
```

Complex objects can be mapped to arguments by specifying each field.

This example maps the `Product.width` and `Product.height` fields to the
`dimension` argument:

```graphql example
type Product {
  shippingCost(
    dimension: DimensionInput @require(field: "{ width: width height: height }")
  ): Currency
}
```

The shorthand equivalent is:

```graphql example
type Product {
  shippingCost(
    dimension: DimensionInput @require(field: "{ width height }")
  ): Currency
}
```

In case the input field names do not match the output field names, explicit
mapping is required.

```graphql example
type Product {
  shippingCost(
    dimension: DimensionInput @require(field: "{ w: width h: height }")
  ): Currency
}
```

Even if `Product.dimension` has all the fields needed for the input object, an
explicit mapping is always required.

This example is NOT allowed because it lacks explicit mapping:

```graphql counter-example
type Product {
  shippingCost(dimension: DimensionInput @require(field: "dimension")): Currency
}
```

Instead, you can traverse into output fields by specifying the path.

This example shows how to map nested fields explicitly:

```graphql example
type Product {
  shippingCost(
    dimension: DimensionInput
      @require(field: "{ width: dimension.width height: dimension.height }")
  ): Currency
}
```

The path does NOT affect the structure of the input object. It is only used to
traverse the output object:

```graphql example
type Product {
  shippingCost(
    dimension: DimensionInput
      @require(field: "{ width: size.width height: size.height }")
  ): Currency
}
```

To avoid repeating yourself, you can prefix the selection with a path that ends
in a dot to traverse INTO the output type.

This affects how fields get interpreted but does NOT affect the structure of the
input object:

```graphql example
type Product {
  shippingCost(
    dimension: DimensionInput @require(field: "dimension.{ width height }")
  ): Currency
}
```

This example is equivalent to the previous one:

```graphql example
type Product {
  shippingCost(
    dimension: DimensionInput @require(field: "size.{ width height }")
  ): Currency
}
```

The path syntax is required for lists because list-valued path expressions would
be ambiguous otherwise.

This example is NOT allowed because it lacks the dot syntax for lists:

```graphql counter-example
type Product {
  shippingCost(
    dimensions: [DimensionInput]
      @require(field: "{ width: dimensions.width height: dimensions.height }")
  ): Currency
}
```

Instead, use the path syntax and brackets to specify the list elements:

```graphql example
type Product {
  shippingCost(
    dimensions: [DimensionInput] @require(field: "dimensions[{ width height }]")
  ): Currency
}
```

With the path syntax it is possible to also select fields from a list of nested
objects:

```graphql example
type Product {
    shippingCost(partIds: @require(field: "parts[id]")): Currency
}
```

For more complex input objects, all these constructs can be nested. This allows
for detailed and precise mappings.

This example nests the `weight` field and the `dimension` object with its
`width` and `height` fields:

```graphql example
type Product {
  shippingCost(
    package: PackageInput
      @require(field: "{ weight, dimension: dimension.{ width height } }")
  ): Currency
}
```

This example nests the `weight` field and the `size` object with its `width` and
`height` fields:

```graphql example
type Product {
  shippingCost(
    package: PackageInput
      @require(field: "{ weight, size: dimension.{ width height } }")
  ): Currency
}
```

The label can be used to nest values that aren't nested in the output.

This example nests `Product.width` and `Product.height` under `dimension`:

```graphql example
type Product {
  shippingCost(
    package: PackageInput
      @require(field: "{ weight, dimension: { width height } }")
  ): Currency
}
```

In the following example, dimensions are nested under `dimension` in the output:

```graphql example
type Product {
  shippingCost(
    package: PackageInput
      @require(field: "{ weight, dimension: dimension.{ width height } }")
  ): Currency
}
```

## Language

According to the GraphQL specification, an argument is a key-value pair in which
the key is the name of the argument and the value is a `Value`.

The `Value` of an argument can take various forms: it might be a scalar value
(such as `Int`, `Float`, `String`, `Boolean`, `Null`, or `Enum`), a list
(`ListValue`), an input object (`ObjectValue`), or a `Variable`.

Within the scope of the {FieldSelectionMap}, the relationship between input and
output is established by defining the `Value` of the argument as a selection of
fields from the output object.

Yet only certain types of `Value` have a semantic meaning. `ObjectValue` and
`ListValue` are used to define the structure of the value. Scalar values, on the
other hand, do not carry semantic importance in this context.

While variables may have legitimate use cases, they are considered out of scope
for the current discussion.

However, it's worth noting that there could be potential applications for
allowing them in the future.

Given that these potential values do not align with the standard literals
defined in the GraphQL specification, a new literal called {SelectedValue} is
introduced, along with {SelectedObjectValue}.

Beyond these literals, an additional literal called {Path} is necessary.

### Name

Is equivalent to the {Name} defined in the
[GraphQL specification](https://spec.graphql.org/October2021/#Name)

### Path

Path ::

- < TypeName > . PathSegment
- PathSegment

PathSegment ::

- FieldName
- FieldName . PathSegment
- FieldName < TypeName > . PathSegment

FieldName ::

- Name

TypeName ::

- Name

The {Path} literal is a string used to select a single output value from the
_return type_ by specifying a path to that value. This path is defined as a
sequence of field names, each separated by a period (`.`) to create segments.

```graphql example
book.title
```

Each segment specifies a field in the context of the parent, with the root
segment referencing a field in the _return type_ of the query. Arguments are not
allowed in a {Path}.

To select a field when dealing with abstract types, the segment selecting the
parent field must specify the concrete type of the field using angle brackets
after the field name if the field is not defined on an interface.

In the following example, the path `mediaById<Book>.isbn` specifies that
`mediaById` returns a `Book`, and the `isbn` field is selected from that `Book`.

```graphql example
mediaById<Book>.isbn
```

### SelectedValue

SelectedValue ::

- SelectedValue | SelectedValueEntry
- `|`? SelectedValueEntry

SelectedValueEntry ::

- Path [lookahead != `.`]
- Path . SelectedObjectValue
- Path SelectedListValue
- SelectedObjectValue

A {SelectedValue} consists of one or more {SelectedValueEntry} components, which
may be joined by a pipe (`|`) operator to indicate alternative selections based
on type.

Each {SelectedValueEntry} may take one of the following forms:

- A {Path} (when not immediately followed by a dot) that is designed to point to
  a single value, although it may reference multiple fields depending on its
  return type.
- A {Path} immediately followed by a dot and a {SelectedObjectValue} to denote a
  nested object selection.
- A {Path} immediately followed by a {SelectedListValue} to denote selection
  from a list.
- A standalone {SelectedObjectValue} 

In the following example, the value could be `title` when referring to a `Book`
and `movieTitle` when referring to a `Movie`.

```graphql example
mediaById<Book>.title | mediaById<Movie>.movieTitle
```

The `|` operator can be used to match multiple possible {SelectedValue}. This
operator is applied when mapping an abstract output type to a `@oneOf` input
type.

```graphql example
{ movieId: <Movie>.id } | { productId: <Product>.id }
```

```graphql example
{ nested: { movieId: <Movie>.id } | { productId: <Product>.id }}
```

### SelectedObjectValue

SelectedObjectValue ::

- { SelectedObjectField+ }

SelectedObjectField ::

- Name: SelectedValue
- Name

{SelectedObjectValue} are unordered lists of keyed input values wrapped in
curly-braces `{}`. It has to be used when the expected input type is an object
type.

This structure is similar to the `ObjectValue` defined in the GraphQL
specification, but it differs by allowing the inclusion of {Path} values within
a {SelectedValue}, thus extending the traditional `ObjectValue` capabilities to
support direct path selections.

A {SelectedObjectValue} following a {Path} is scoped to the type of the field
selected by the {Path}. This means that the root of all {SelectedValue} inside
the selection is no longer scoped to the root (defined by `@is` or `@require`)
but to the field selected by the {Path}. The {Path} does not affect the
structure of the input type.

This allows for reducing repetition in the selection.

The following example is valid:

```graphql example
type Product {
  dimension: Dimension!
  shippingCost(
    dimension: DimensionInput! @require(field: "dimension.{ size weight }")
  ): Int!
}
```

The following example is equivalent to the previous one:

```graphql example
type Product {
  dimensions: Dimension!
  shippingCost(
    dimensions: DimensionInput!
      @require(field: "{ size: dimensions.size weight: dimensions.weight }")
  ): Int! @lookup
}
```

### SelectedListValue

SelectedListValue ::

- [ SelectedValue ]
- [ SelectedListValue ]

A {SelectedListValue} is an ordered list of {SelectedValue} wrapped in square
brackets `[]`. It is used to express semantic equivalence between an argument
expecting a list of values and the values of a list field within the output
object.

The {SelectedListValue} differs from the `ListValue` defined in the GraphQL
specification by only allowing one {SelectedValue} as an element.

The following example is valid:

```graphql example
type Product {
  parts: [Part!]!
  partIds(partIds: [ID!]! @require(field: "parts[id]")): [ID!]!
}
```

In this example, the `partIds` argument is semantically equivalent to the `id`
fields of the `parts` list.

The following example is invalid because it uses multiple {SelectedValue} as
elements:

```graphql counter-example
type Product {
  parts: [Part!]!
  partIds(parts: [PartInput!]! @require(field: "parts[id name]")): [ID!]!
}

input PartInput {
  id: ID!
  name: String!
}
```

A {SelectedObjectValue} can be used as an element of a {SelectedListValue} to
select multiple object fields as long as the input type is a list of
structurally equivalent objects.

Similar to {SelectedObjectValue}, a {SelectedListValue} following a {Path} is
scoped to the type of the field selected by the {Path}. This means that the root
of all {SelectedValue} inside the selection is no longer scoped to the root
(defined by `@is` or `@require`) but to the field selected by the {Path}. The
{Path} does not affect the structure of the input type.

The following example is valid:

```graphql example
type Product {
  parts: [Part!]!
  partIds(parts: [PartInput!]! @require(field: "parts[{ id name }]")): [ID!]!
}

input PartInput {
  id: ID!
  name: String!
}
```

In case the input type is a nested list, the shape of the input object must
match the shape of the output object.

```graphql example
type Product {
  parts: [[Part!]]!
  partIds(
    parts: [[PartInput!]]! @require(field: "parts[[{ id name }]]")
  ): [ID!]!
}

input PartInput {
  id: ID!
  name: String!
}
```

The following example is valid:

```graphql example
type Query {
  findLocation(
    location: LocationInput!
      @is(field: "{ coordinates: coordinates[{lat: x lon: y}]}")
  ): Location @lookup
}

type Coordinate {
  x: Int!
  y: Int!
}

type Location {
  coordinates: [Coordinate!]!
}

input PositionInput {
  lat: Int!
  lon: Int!
}

input LocationInput {
  coordinates: [PositionInput!]!
}
```

## Validation

Validation ensures that {FieldSelectionMap} scalars are semantically correct
within the given context.

Validation of {FieldSelectionMap} scalars occurs during the composition phase,
ensuring that all {FieldSelectionMap} entries are syntactically correct and
semantically meaningful relative to the context.

Composition is only possible if the {FieldSelectionMap} is validated
successfully. An invalid {FieldSelectionMap} results in undefined behavior,
making composition impossible.

In this section, we will assume the following type system in order to
demonstrate examples:

```graphql
type Query {
  mediaById(mediaId: ID!): Media
  findMedia(input: FindMediaInput): Media
  searchStore(search: SearchStoreInput): [Store]!
  storeById(id: ID!): Store
}

type Store {
  id: ID!
  city: String!
  media: [Media!]!
}

interface Media {
  id: ID!
}

type Book implements Media {
  id: ID!
  title: String!
  isbn: String!
  author: Author!
}

type Movie implements Media {
  id: ID!
  movieTitle: String!
  releaseDate: String!
}

type Author {
  id: ID!
  books: [Book!]!
}

input FindMediaInput @oneOf {
  bookId: ID
  movieId: ID
}

type SearchStoreInput {
  city: String
  hasInStock: FindMediaInput
}
```

### Path Field Selections

Each segment of a {Path} must correspond to a valid field defined on the current
type context.

**Formal Specification**

- For each {segment} in the {Path}:
  - If the {segment} is a field
    - Let {fieldName} be the field name in the current {segment}.
    - {fieldName} must be defined on the current type in scope.

**Explanatory Text**

The {Path} literal is used to reference a specific output field from a input
field. Each segment in the {Path} must correspond to a field that is valid
within the current type scope.

For example, the following {Path} is valid in the context of `Book`:

```graphql example
title
```

```graphql example
<Book>.title
```

Incorrect paths where the field does not exist on the specified type is not
valid result in validation errors. For instance, if `<Book>.movieId` is
referenced but `movieId` is not a field of `Book`, will result in an invalid
{Path}.

```graphql counter-example
movieId
```

```graphql counter-example
<Book>.movieId
```

### Path Terminal Field Selections

Each terminal segment of a {Path} must follow the rules regarding whether the
selected field is a leaf node.

**Formal Specification**

- For each {segment} in the {Path}:
  - Let {selectedType} be the unwrapped type of the current {segment}.
  - If {selectedType} is a scalar or enum:
    - There must not be any further segments in {Path}.
  - If {selectedType} is an object, interface, or union:
    - There must be another segment in {Path}.

**Explanatory Text**

A {Path} that refers to scalar or enum fields must end at those fields. No
further field selections are allowed after a scalar or enum. On the other hand,
fields returning objects, interfaces, or unions must continue to specify further
selections until you reach a scalar or enum field.

For example, the following {Path} is valid if `title` is a scalar field on the
`Book` type:

```graphql example
book.title
```

The following {Path} is invalid because `title` should not have subselections:

```graphql counter-example
book.title.something
```

For non-leaf fields, the {Path} must continue to specify subselections until a
leaf field is reached:

```graphql example
book.author.id
```

Invalid {Path} where non-leaf fields do not have further selections:

```graphql counter-example
book.author
```

### Type Reference Is Possible

Each segment of a {Path} that references a type, must be a type that is valid in
the current context.

**Formal Specification**

- For each {segment} in a {Path}:
  - If {segment} is a type reference:
    - Let {type} be the type referenced in the {segment}.
    - Let {parentType} be the type of the parent of the {segment}.
    - Let {applicableTypes} be the intersection of {GetPossibleTypes(type)} and
      {GetPossibleTypes(parentType)}.
    - {applicableTypes} must not be empty.

GetPossibleTypes(type):

- If {type} is an object type, return a set containing {type}.
- If {type} is an interface type, return the set of types implementing {type}.
- If {type} is a union type, return the set of possible types of {type}.

**Explanatory Text**

Type references inside a {Path} must be valid within the context of the
surrounding type. A type reference is only valid if the referenced type could
logically apply within the parent type.

### Values of Correct Type

**Formal Specification**

- For each SelectedValue {value}:
  - Let {type} be the type expected in the position {value} is found.
  - {value} must be coercible to {type}.

**Explanatory Text**

Literal values must be compatible with the type expected in the position they
are found.

The following examples are valid use of value literals in the context of
{FieldSelectionMap} scalar:

```graphql example
type Query {
  storeById(id: ID! @is(field: "id")): Store! @lookup
}

type Store {
  id: ID
  city: String!
}
```

Non-coercible values are invalid. The following examples are invalid:

```graphql counter-example
type Query {
  storeById(id: ID! @is(field: "id")): Store! @lookup
}

type Store {
  id: Int
  city: String!
}
```

### Selected Object Field Names

**Formal Specification**

- For each Selected Object Field {field} in the document:
  - Let {fieldName} be the Name of {field}.
  - Let {fieldDefinition} be the field definition provided by the parent
    selected object type named {fieldName}.
  - {fieldDefinition} must exist.

**Explanatory Text**

Every field provided in an selected object value must be defined in the set of
possible fields of that input object's expected type.

For example, the following is valid:

```graphql example
type Query {
  storeById(id: ID! @is(field: "id")): Store! @lookup
}

type Store {
  id: ID
  city: String!
}
```

In contrast, the following is invalid because it uses a field "address" which is
not defined on the expected type:

```graphql counter-example
extend type Query {
  storeById(id: ID! @is(field: "address")): Store! @lookup
}

type Store {
  id: ID
  city: String!
}
```

### Selected Object Field Uniqueness

**Formal Specification**

- For each selected object value {selectedObject}:
  - For every {field} in {selectedObject}:
    - Let {name} be the Name of {field}.
    - Let {fields} be all Selected Object Fields named {name} in
      {selectedObject}.
    - {fields} must be the set containing only {field}.

**Explanatory Text**

Selected objects must not contain more than one field with the same name, as it
would create ambiguity and potential conflicts.

For example, the following is invalid:

```graphql counter-example
extend type Query {
  storeById(id: ID! @is(field: "id id")): Store! @lookup
}

type Store {
  id: ID
  city: String!
}
```

### Required Selected Object Fields

**Formal Specification**

- For each Selected Object:
  - Let {fields} be the fields provided by that Selected Object.
  - Let {fieldDefinitions} be the set of input object field definitions of that
    Selected Object.
  - For each {fieldDefinition} in {fieldDefinitions}:
    - Let {type} be the expected type of {fieldDefinition}.
    - Let {defaultValue} be the default value of {fieldDefinition}.
    - If {type} is Non-Null and {defaultValue} does not exist:
      - Let {fieldName} be the name of {fieldDefinition}.
      - Let {field} be the input object field in {fields} named {fieldName}.
      - {field} must exist.

**Explanatory Text**

Input object fields may be required. This means that a selected object field is
required if the corresponding input field is required. Otherwise, the selected
object field is optional.

For instance, if the `UserInput` type requires the `id` field:

```graphql example
input UserInput {
  id: ID!
  name: String!
}
```

Then, an invalid selection would be missing the required `id` field:

```graphql counter-example
extend type Query {
  userById(user: UserInput! @is(field: "{ name: name }")): User! @lookup
}
```

If the `UserInput` type requires the `name` field, but the `User` type has an
optional `name` field, the following selection would be valid.

```graphql example
extend type Query {
  findUser(input: UserInput! @is(field: "{ name: name }")): User! @lookup
}

type User {
  id: ID
  name: String
}

input UserInput {
  id: ID
  name: String!
}
```

But if the `UserInput` type requires the `name` field but it's not defined in
the `User` type, the selection would be invalid.

```graphql counter-example
extend type Query {
  findUser(input: UserInput! @is(field: "{ id: id }")): User! @lookup
}

type User {
  id: ID
}

input UserInput {
  id: ID
  name: String!
}
```
