# Appendix A: Specification of FieldSelection Scalar

## Introduction

This appendix focuses on the specification of the {FieldSelection} scalar type. {FieldSelection} is
designed to express semantic equivalence between arguments of a field and fields within the result 
type. Specifically, it allows defining complex relationships between input arguments and fields in
the output object by encapsulating these relationships within a parsable string format. It is used
in the `@is` and `@requires` directives.


To illustrate, consider a simple example from a GraphQL schema:

```graphql
type Query {
    userById(userId: ID! @is(field: "id")): User! @lookup
}
```

In this schema, the `userById` query uses the `@is` directive with {FieldSelection} to declare that
the `userId` argument is semantically equivalent to the `User.id` field. 

An example query might look like this:

```graphql
query {
    userById(userId: "123") {
        id
    }
}
```

Here, it is exptected that the `userId` "123" corresponds directly to `User.id`, resulting in the
following response if correctly implemented:

```json
{
    "data": {
        "userById": {
            "id": "123"
        }
    }
}
```

The {FieldSelection} scalar type is used to establish semantic equivalence between an argument and
the fields within the associated return type. To accomplish this, the scalar must define the
relationship between input fields or arguments and the fields in the resulting object. 
Consequently, a {FieldSelection} only makes sense in the context of a specific argument and its 
return type.

The {FieldSelection} scalar is represented as a string that, when parsed, produces a {SelectedValue}.

A {SelectedValue} must exactly match the shape of the argument value to be considered
valid. For non-scalar arguments, you must specify each field of the input type in
{SelectedObjectValue}.

```graphql example
extend type Query {
    findUserByName(user: UserInput! @is(field: "{ firstName: firstName }")): User @lookup
}
```

```graphql counter-example
extend type Query {
    findUserByName(user: UserInput! @is(field: "firstName")): User @lookup
}
```


## Language

According to the GraphQL specification, an argument is a key-value pair in which the key is the name
of the argument and the value is a `Value`.

The `Value` of an argument can take various forms: it might be a scalar value (such as `Int`,
`Float`, `String`, `Boolean`, `Null`, or `Enum`), a list (`ListValue`), an input object
(`ObjectValue`), or a `Variable`.

Within the scope of the {FieldSelection}, the relationship between input and output is
established by defining the `Value` of the argument as a selection of fields from the output object.

Yet only certain types of `Value` have a semantic meaning. 
`ObjectValue` and `ListValue` are used to define the structure of the value. 
Scalar values, on the other hand, do not carry semantic importance in this context, and variables
are excluded as they do not exist. 
Given that these potential values do not align with the standard literals defined in the GraphQL
specification, a new literal called {SelectedValue} is introduced, along with {SelectedObjectValue},

Beyond these literals, an additional literal called {Path} is necessary.

### Path
Path :: 
    - Name
    - Path . Path
    - Path | Path
    - Name < Name > . Path

The {Path} literal is a string used to select a single output value from the _return type_ by
specifying a path to that value. 
This path is defined as a sequence of field names, each separated by a period (`.`) to create 
segments. 

``` example
book.title
```

Each segment specifies a field in the context of the parent, with the root segment referencing a
field in the _return type_ of the query. 
Arguments are not allowed in a {Path}.

To select a field when dealing with abstract types, the segment selecting the parent field must 
specify the concrete type of the field using angle brackets after the field name if the field is not 
defined on an interface.

In the following example, the path `mediaById.<Book>.isbn` specifies that `mediaById` returns a
`Book`, and the `isbn` field is selected from that `Book`.

``` example
mediaById<Book>.isbn
```

A {Path} is designed to point to only a single value, although it may reference multiple fields
depending on the return type. To allow selection from different paths based on
type, a {Path} can include multiple paths separated by a pipe (`|`), such as in

In the following example, the value could be `title` when referring to a `Book` and `movieTitle`
when referring to a `Movie`. 

``` example
mediaById<Book>.title | mediaById<Movie>.movieTitle
```

### SelectedValue
SelectedValue :: 
    - Path
    - SelectedObjectValue

A {SelectedValue} is defined as either a {Path} or a {SelectedObjectValue}

### SelectedObjectValue
SelectedObjectValue :: 
    - { SelectedObjectField+ }

SelectedObjectField :: 
    - Name: SelectedValue

{SelectedObjectValue} are unordered lists of keyed input values wrapped in curly-braces `{}`. 
This structure is similar to the `ObjectValue` defined in the GraphQL specification, but it
differs by allowing the inclusion of {Path} values within a {SelectedValue}, thus extending the
traditional `ObjectValue` capabilities to support direct path selections.

### Name
Is equivalent to the `Name` defined in the GraphQL specification.


