This RFC proposes a syntax for `FieldSelection` that allows for flexible field selections and reshaping of input objects.

# Motivation

The directive `@is` specifies semantic equivalence between an argument and fields within the resulting type.
The argument named `field` accepts a string which adheres to a specific format defined by the scalar type `FieldSelection`.

For instance, consider the following field definition:
```graphql
type Query {
    userById(userId: ID! @is(field: "id")): User! @lookup
}
```

In this example, the semantic equivalence is established between the argument `userId` and the field `User.id`.
This equivalence instructs the system on composition, validation, and execution to treat `userId` as semantically identical to `User.id`.

Consider the execution of a query as follows:
```graphql
query {
    userById(userId: "123") {
        id
    }
}
```
In this scenario, the only correct response that does not result in an error would be:
```json
{
    "data": {
        "userById": {
            "id": "123"
        }
    }
}
```

The scalar `FieldSelection` is similarly used in the `@requires` directive, despite its use in different contexts.
This document aims to explore various cases where `FieldSelection` is used and to discuss potential solutions to challenges presented by its use.

# Cases
This section outlines various scenarios that need to be addressed.
It is intended to describe the problem cases and is not focused on providing solutions.

## Single field 
This subsection addresses cases involving a single field reference.
It details the simplest scenario where a single field must be referenced, establishing semantic equivalence between an argument and a corresponding field within a type.

In the following example we specifcy the semantic equivalence between the argument `id` and the field `User.id`.
```graphql
extend type Query {
    userById(id: ID!): User @lookup
}
```

### Single Field - Fieldname Matches
In the simplest scenario, the field name in the argument matches the field name in the returned type.
For example, the argument `id` is semantically equivalent to the field `User.id`.

```graphql
extend type Query {
    userById(id: ID!): User @lookup
}

type User {
    id: ID!
}
```

### Single Field - Fieldname is Different
In some instances, the field name in the returned type differs from the name of the argument.
In this scenario, the argument `userId` is semantically equivalent to the field `User.id`.

```graphql
extend type Query {
    userById(userId: ID!): User @lookup
}

type User {
    id: ID!
}
```

### Single Field - Field is Deeper in the Tree
This case addresses situations where the field is nested deeper within a type's structure. 
Here, the argument `addressId` is semantically equivalent to the field `User.address.id`.

```graphql
extend type Query {
    userByAddressId(id: ID!): User @lookup
}
type User {
    id: ID!
    address: Address
}
type Address {
    id: ID!
}
```

### Single Field - Field is Deeper in the Tree and the Fieldname is Different
This case is similar to the previous, but involves a different field name, where the argument `addressId` is semantically equivalent to `User.address.id`.

```graphql
extend type Query {
    userByAddressId(addressId: ID!): User @lookup
}

type User {
    id: ID!
    address: Address
}

type Address {
    id: ID!
}
```

### Single Field - Abstract Type - Field Matches Name
In cases involving abstract types, the argument may correspond to fields in different concrete types but with matching field names.
For instance, the argument `id` could be semantically equivalent to either `Movie.id` or `Book.id`.

```graphql
extend type Query {
   mediaById(id: ID!): MovieOrBook @lookup
}

type Movie {
    id: ID!
}

type Book {
    id: ID!
}

union MovieOrBook = Movie | Book
```

### Single Field - Abstract Type - Field is Different
In this variation involving abstract types, the field names differ across implementations. 
Here, the argument `id` corresponds semantically to `Movie.movieId` or `Book.bookId`.

```graphql
extend type Query {
   mediaById(id: ID!): MovieOrBook @lookup
}

type Movie {
    movieId: ID!
}

type Book {
    bookId: ID!
}

union MovieOrBook = Movie | Book
```

## Multiple Field Reference
Another scenario is when a field references multiple fields within the returned type.
This is particularly useful when the returning type has a composite key and the argument is an input type.

### Multiple Field Reference - Fieldname Matches
In this scenario, the input fields `UserInput.firstName` and `UserInput.lastName` are semantically equivalent to the fields `User.firstName` and `User.lastName`.

```graphql
type Query {
    findUserByName(user: UserInput!): User @lookup
}

input UserInput {
    firstName: String!
    lastName: String!
}

type User {
    firstName: String!
    lastName: String!
}
```

### Multiple Field Reference - Fieldname is Different
Here, the input fields `UserInput.firstName` and `UserInput.lastName` are semantically equivalent to the fields `User.givenName` and `User.familyName`.

```graphql
type Query {
    findUserByName(user: UserInput!): User @lookup
}

input UserInput {
    firstName: String!
    lastName: String!
}

type User {
    givenName: String!
    familyName: String!
}
```

### Multiple Field Reference - Output field is Deeper in the Tree
This scenario involves the input fields `UserInput.firstName` and `UserInput.lastName` being semantically equivalent to the fields `Profile.firstName` and `User.lastName`.

```graphql
type Query {
    findUserByName(user: UserInput!): User @lookup
}

input UserInput {
    firstName: String!
    lastName: String!
}

type User {
    profile: Profile
    lastName: String!
}

type Profile {
    firstName: String!
}
```

### Multiple Field Reference - Output field is Deeper in the Tree with Abstract Type
In this case, the input fields `UserInput.firstName` and `UserInput.lastName` correspond to either `EntraProfile.firstName` or `AdfsProfile.firstName`, and `User.lastName`.

```graphql
type Query {
    findUserByName(user: UserInput!): User @lookup
}
input UserInput {
    firstName: String!
    lastName: String!
}

type User {
    profile: Profile
    lastName: String!
}

union Profile = EntraProfile | AdfsProfile

type EntraProfile {
    firstName: String!
}

type AdfsProfile {
    name: String!
}
```

### Multiple Field Reference - Output field is Deeper in the Tree with Abstract Type and Fieldname is Different
Here, the input fields `UserInput.firstName` and `UserInput.lastName` correspond to either `EntraProfile.name` or `AdfsProfile.userName`, and `User.lastName`.

```graphql
type Query {
    findUserByName(user: UserInput!): User @lookup
}

input UserInput {
    firstName: String!
    lastName: String!
}

type User {
    profile: Profile
    lastName: String!
}

union Profile = EntraProfile | AdfsProfile

type EntraProfile {
    name: String!
}

type AdfsProfile {
    userName: String!
}
```

### Multiple Field Reference - Input field is Deeper in the Tree
This introduces additional complexity as the input object requires reshaping.
The input fields `UserInput.info.firstName` and `UserInput.name` correspond to the fields `User.firstName` and `User.name`.

```graphql
type Query {
    findUserByName(user: UserInput!): User @lookup
}

input UserInput {
    info: Info
    name: String!
}

type Info {
    firstName: String!
e

type User {
    firstName: String!
    name: String!
}
```

### Multiple Field Reference - Input field is Deeper in the Tree with Abstract Type
In this instance, the input fields `UserInput.info.firstName` and `UserInput.name` correspond to either `EntraUser.firstName` or `AdfsUser.lastName`, and `User.name`.

```graphql
type Query {
    findUserByName(user: UserInput!): User @lookup
}

input UserInput {
    info: Info
    name: String!
}

type Info {
    firstName: String!
}

interface User {
    name: String!
}

type EntraUser implements User {
    firstName: String!
}

type AdfsUser implements User {
    lastName: String!
}
```

### Multiple Field Reference - Input field is Deeper in the Tree and Output field is Deeper in the Tree
Here, the input fields `UserInput.info.firstName` and `UserInput.name` correspond to the fields `User.profile.firstName` and `User.name`.

```graphql
type Query {
    findUserByName(user: UserInput!): User @lookup
}

input UserInput {
    info: Info
    name: String!
}

type Info {
    firstName: String!
}

type User {
    profile: Profile
    name: String!
}

type Profile {
    firstName: String!
}
```

### Multiple Field Reference - Input field is Deeper in the Tree and Output field is Deeper in the Tree with Abstract Type
Finally, the input fields `UserInput.info.firstName` and `UserInput.name` correspond to the fields `EntraProfile.firstName` and `AdfsProfile.name`, and `User.name`.

```graphql
type Query {
    findUserByName(user: UserInput!): User @lookup
}

input UserInput {
    info: Info
    name: String!
}

type Info {
    firstName: String!
}

type User {
    profile: Profile
    name: String!
}

union Profile = EntraProfile | AdfsProfile

type EntraProfile {
    firstName: String!
}

type AdfsProfile {
    name: String!
}
```

## Edge Cases - Input Field Shares Name with Argument and Output Field

This subsection explores a the case where the input field `UserInput.id` is semantically equivalent to the field `User.id`.
Such a scenario introduces ambiguity because the same field name `id` is used across different contexts, which current solutions struggle to express clearly.

```graphql
type Query {
    userById(id: InputObject @is(field: "id")): User @lookup
}

input InputObject {
    id: ID!
}

type User {
    id: ID!
}
```

# Different Possible Solutions

There are numerous potential solutions to address the challenges presented by `FieldSelection` however, each solution introduces its own set of complexities.

## FieldSelection as a selection set

The `FieldSelection` scalar extends the concept of a selection set, allowing for flexible field selections.

In the simplest form, a single field reference might appear as follows:

**Single Field Reference**
```graphql
extend type Query {
    userById(id: ID! @is(field: "id")): User @lookup
}
```

A more complex scenario involves multiple fields:

**Multiple Field Reference**
```graphql
extend type Query {
    findUserByName(user: UserInput! @is(field: "firstName lastName")): User @lookup
}
```

Abstract fields across different types can also be specified easily:

**Abstract Field Reference**
```graphql
extend type Query {
   mediaById(id: ID! @is(field: "... Movie { id } ... Book { id }")): MovieOrBook @lookup
}
```

For fields nested deeper in a type's hierarchy:

**Single Field Nested Reference**
```graphql
extend type Query {
    userById(userId: ID! @is(field: "address { id }")): User @lookup
}
```

This notation implies that the nested field `id` under `address` is semantically equivalent to the argument `userId`.

However, complexities come when multiple fields need to be referenced, particularly when they are nested:

**Multiple Field Reference - Output field is deeper in the tree**
```graphql
type Query {
    findUserByName(user: UserInput! @is(field: "profile { firstName } lastName")): User @lookup
}

input UserInput {
    firstName: String!
    lastName: String!
}
```

The challenge here is to clearly define that `User.profile.firstName` is equivalent to `UserInput.firstName`.

To address this, a notation like input object fields could be utilized:
```graphql
type Query {
    findUserByName(user: UserInput! @is(field: "firstName: { profile { firstName } } lastName")): User @lookup
}
```
However, this solution mixes input fields and output fields, potentially leading to confusion in
interpreting the mappings.

## `FieldSelection` as a path combined with an input object
Another solution is to treat the `FieldSelection` as a path and borrow a few concepts from the input object syntax to shape the input object.

A single field reference would look exactly the same as in the previous example. 

**Single Field Reference**
```graphql
extend type Query {
    userById(id: ID! @is(field: "id")): User @lookup
}
```

Multiple field references follow the same format:
**Multiple Field Reference**
```graphql
extend type Query {
    findUserByName(user: UserInput! @is(field: "firstName lastName")): User @lookup
}
```

To reference a deeper nested field, a path syntax is used:
**Single Field Nested Reference**
```graphql
extend type Query {
    userById(userId: ID! @is(field: "address.id")): User @lookup
}
```

This approach also allows for pointing to fields across different hierarchies:

**Multiple Field Reference - Output field is deeper in the tree**
```graphql
type Query {
    findUserByName(user: UserInput! @is(field: "profile.firstName lastName")): User @lookup
}

input UserInput {
    firstName: String!
    lastName: String!
}
```

To handle renaming, simply specify the new name followed by the path to the field:
```graphql
type Query {
    findUserByName(user: UserInput! @is(field: "firstName: profile.name lastName")): User @lookup
}
```

The main challenge with this approach is abstract types. A generic path syntax might solve this for some cases:

**Output field is deeper in the tree with abstract type and fieldname is different**
```graphql
extend type Query {
   findUserByName(user: UserInput! @is(field: "firstName: profile<EntraProfile>.name lastName")): User @lookup
}
```

However, it fails when multiple abstract types are possible for the same field unless we introduce a
new syntax to handle this:

```graphql
extend type Query {
   mediaById(id: ID! @is(field: "<Movie>.id | <Book>.id")): MovieOrBook @lookup
}
```


## `FieldSelection` Inline Selection

When evaluating the challenges of previous approaches:
- The first approach effectively handles abstract types using selection sets equal to query
operations, but struggles with reshaping the input object due to the limitations of the selection
set syntax.
- The second approach excels in reshaping the input object by specifying exact fields, thus avoiding
the ambiguities of selection sets, but it falls short in handling abstract types due to the
constraints of the path syntax.

To combine the best of both worlds we use the input object like syntax of the path syntax and
combine it with the selection set syntax of the first approach. 

### Concept Overview
This outline is not a formal specification but aims to clarify the concept:
1. A `FieldSelection` consists of `Selection`s and `InputField`s:
   - A `Selection` targets a single field within the selection set of the return type:
     - The simplest form of a `Selection` is an `ObjectField` that references a field within the selection set of the returning type.
     - A `Selection` inherently acts as an `InputField` with the same name as the field it selects:
       - If multiple field names are possible, the `Selection` must be nested within an `InputField`.
     - Selecting multiple fields within a single `Selection` is prohibited to prevent ambiguity.
     - Aliases are not supported within a `Selection`.
     - Arguments are not supported within a `Selection`.
     - Directives are not supported within a `Selection`.
   - An `InputField` corresponds directly to a field within the input object:
     - It is a key-value pair where the key is the field name and the value is either an `ObjectField` or a `Selection` -> `fieldName: value`.
       - If the value of an `ObjectField` begins with a `{`, it is classified as an `ObjectValue`.
       - Otherwise, it is treated as a `Selection`.
   - An `ObjectValue` is an aggregate of `InputField`s.

So simple field selection would look like this:
**Single Field Reference**
```graphql
extend type Query {
    userById(id: ID! @is(field: "id")): User @lookup
}
```

and multiple field selection would look like this:
**Multiple Field Reference**
```graphql
extend type Query {
    findUserByName(user: UserInput! @is(field: "firstName lastName")): User @lookup
}
```

if you need to rename a field you just use a `InputField` and set the value to an `ObjectField`

**Multiple Field Reference - Fieldname is different**
```graphql
extend type Query {
    #                   Selection:                         +--------+           +---------+
    #                   InputField:            +---------+          +---------+
    findUserByName(user: UserInput! @is(field: "firstName: givenName lastName: familyName")): User @lookup
}
```

You can easily express abstract types as well

**Abstract Field Reference**
```graphql
extend type Query {
   mediaById(id: ID! @is(field: "... Movie { id } ... Book { id }")): MovieOrBook @lookup
}
```

but you can also shape the input object

**Multiple Field Reference - Output field is deeper in the tree**
```graphql
type Query {
    #
    #                       Selection:                    +--------------------+ +--------+
    #                       InputField:        +---------+
    findUserByName(user: UserInput! @is(field: "firstName: profile { firstName } lastName")): User @lookup
}
```

you can use abstract types as well

**Multiple Field Reference - Output field is deeper in the tree with abstract type and fieldname is different**
```graphql
    #
    #                       Selection:                    +------------------------------------------------------------------------+ +--------+
    #                       InputField:        +---------+
    findUserByName(user: UserInput! @is(field: "firstName: profile { ... on EntraProfile { name } ... on AdfsProfile { firstName } } lastName")): User @lookup
```

or even nest the input object

**Multiple Field Reference - Input field is deeper in the tree and output field is deeper in the tree with abstract type**
```graphql
    #
    #                       Selection:                               +-----------------------------------------------------------------------+   +--------+
    #                       InputField:        +---------+
    #                       ObjectValue:                 +---------+                                                                          +-+
    findUserByName(user: UserInput! @is(field: "profile: {firstName: profile { ... on EntraProfile { name } ... on AdfsProfile { firstName } } } lastName")): User @lookup
```

# Overview 
<table>
<tr>
<td> Name </td> <td> Query </td> <td> Solution1 </td> <td> Solution2 </td> <td> Solution3 </td>
</tr>





<tr>
<td>
Single Field - Fieldname Matches
</td>
<td>

```graphql
extend type Query {
    userById(id: ID!): User @lookup
}

type User {
    id: ID!
}
```

</td>

<td>

```graphql
@is(field: "id")
```

</td>

<td>

```graphql
@is(field: "id")
```

</td>
<td>

```graphql
@is(field: "id")
```

</td>

</tr>






<tr>
<td>
Single Field - Fieldname is different
</td>
<td>

```graphql
extend type Query {
    userById(userId: ID!): User @lookup
}

type User {
    id: ID!
}
```

</td>

<td>

```graphql
@is(field: "id")
```

</td>

<td>

```graphql
@is(field: "id")
```

</td>
<td>

```graphql
@is(field: "id")
```

</td>

</tr>







<tr>
<td>
Single Field - Field is deeper in the tree
</td>
<td>

```graphql
extend type Query {
    userByAddressId(id: ID!): User @lookup
}
type User {
    id: ID!
    address: Address
}
type Address {
    id: ID!
}
```

</td>

<td>

```graphql
@is(field: "address { id }")
```

</td>

<td>

```graphql
@is(field: "address.id")
```

</td>
<td>

```graphql
@is(field: "address { id }")
```

</td>

</tr>





<tr>
<td>
Single Field - Field is deeper in the tree and the fieldname is different
</td>
<td>

```graphql
extend type Query {
    userByAddressId(addressId: ID!): User @lookup
}

type User {
    id: ID!
    address: Address
}

type Address {
    addressId: ID!
}
```

</td>

<td>

```graphql
@is(field: "address { id }")
```

</td>

<td>

```graphql
@is(field: "address.id")
```

</td>
<td>

```graphql
@is(field: "address { id }")
```

</td>

</tr>






<tr>
<td>
Single Field - Abstract Type - Field Matches Name
</td>
<td>

```graphql
extend type Query {
   mediaById(id: ID! ): MovieOrBook @lookup
}

type Movie {
    id: ID!
}

type Book {
    id: ID!
}

union MovieOrBook = Movie | Book
```

</td>

<td>

```graphql
@is(field: "... Movie { id } ... Book { id }")
```

</td>

<td>

```graphql
@is(field: "<Movie>.id | <Book>.id")
```

</td>
<td>

```graphql
@is(field: "... Movie { id } ... Book { id }")j
```
</td>
</tr>

<tr>
<td>
Single Field - Abstract Type - Field is different
</td>
<td>

```graphql
extend type Query {
   mediaById(id: ID! ): MovieOrBook @lookup
}

type Movie {
    movieId: ID!
}

type Book {
    bookId: ID!
}

union MovieOrBook = Movie | Book
```

</td>

<td>

```graphql
@is(field: "... Movie { movieId } ... Book { bookId }")
```

</td>

<td>

```graphql
@is(field: "<Movie>.movieid | <Book>.bookId")
```

</td>
<td>

```graphql
@is(field: "... Movie { movieId } ... Book { bookId }")
```
</td>
</tr>





<tr>
<td>
Multiple Field Reference - Fieldname Matches
</td>
<td>

```graphql
type Query {
    findUserByName(user: UserInput!): User @lookup
}

input UserInput {
    firstName: String!
    lastName: String!
}

type User {
    firstName: String!
    lastName: String!
}
```

</td>

<td>

```graphql
@is(field: "firstName lastName")
```

</td>

<td>

```graphql
@is(field: "firstName lastName")
```

</td>
<td>

```graphql
@is(field: "firstName lastName")
```
</td>
</tr>




<tr>
<td>
Multiple Field Reference - Fieldname is different
</td>
<td>

```graphql
type Query {
    findUserByName(user: UserInput!): User @lookup
}

input UserInput {
    firstName: String!
    lastName: String!
}

type User {
    givenName: String!
    familyName: String!
}
```

</td>

<td>

```graphql
@is(field: "firstName: givenName lastName: familyName")
```

</td>

<td>

```graphql
@is(field: "firstName: givenName lastName: familyName")
```

</td>
<td>

```graphql
@is(field: "firstName: givenName lastName: familyName")
```
</td>
</tr>





<tr>
<td>
Multiple Field Reference - Output field is deeper in the tree 
</td>
<td>

```graphql
type Query {
    findUserByName(user: UserInput!): User @lookup
}

input UserInput {
    firstName: String!
    lastName: String!
}

type User {
    profile: Profile
    lastName: String!
}

type Profile {
    firstName: String!
}
```

</td>

<td>

🚨 This is ambigious

```graphql
@is(field: "profile { firstName } lastName")
```

</td>

<td>

```graphql
@is(field: "profile.firstName lastName")
```

</td>
<td>

```graphql
@is(field: "profile { firstName } lastName")
```
</td>
</tr>



<tr>
<td>
Multiple Field Reference - Output field is deeper in the tree with abstract type
</td>
<td>

```graphql
type Query {
    findUserByName(user: UserInput!): User @lookup
}
input UserInput {
    firstName: String!
    lastName: String!
}

type User {
    profile: Profile
    lastName: String!
}

union Profile = EntraProfile | AdfsProfile

type EntraProfile {
    firstName: String!
}

type AdfsProfile {
    name: String!
}
```

</td>

<td>

🚨 This is ambigious

```graphql
@is(field: "profile { ... on EntraProfile { firstName } ... on AdfsProfile { firstName } } lastName")
```

</td>

<td>

```graphql
@is(field: "profile<EntraProfile>.firstName | profile<AdfsProfile>.firstName lastName")
```

</td>
<td>

```graphql
@is(field: "profile { ... on EntraProfile { firstName } ... on AdfsProfile { firstName} } lastName")
```
</td>
</tr>

<tr>
<td>
Multiple Field Reference - Output field is deeper in the tree with abstract type and fieldname is different
</td>
<td>

```graphql
type Query {
    findUserByName(user: UserInput!): User @lookup
}

input UserInput {
    firstName: String!
    lastName: String!
}

type User {
    profile: Profile
    lastName: String!
}

union Profile = EntraProfile | AdfsProfile
```

</td>
<td>

🚨 This is ambigious

```graphql
@is(field: "firstName: { profile { ... on EntraProfile { name } ... on AdfsProfile { userName } } lastName")
```

</td>

<td>

```graphql
@is(field: "firstName: profile<EntraProfile>.name | profile<AdfsProfile>.userName lastName")
```

</td>
<td>

```graphql
@is(field: "firstName: profile { ... on EntraProfile { name } ... on AdfsProfile { userName } lastName")
```
</td>
</tr>

<tr>
<td>
Multiple Field Reference - Input field is deeper in the tree 
</td>
<td>

```graphql
type Query {
    findUserByName(user: UserInput!): User @lookup
}

input UserInput {
    info: Info
    name: String!
}

type Info {
    firstName: String!
}

type User {
    firstName: String!
    name: String!
}
```

</td>
<td>

🚨 There is no reasonable way to express this 

</td>

<td>

```graphql
@is(field: "profile: { firstName: firstName} lastName")
```

</td>
<td>

```graphql
@is(field: "profile: { firstName: firstName} lastName")
```
</td>
</tr>


<tr>
<td>
Multiple Field Reference - Input field is deeper in the tree with abstract type
</td>
<td>

```graphql
type Query {
    findUserByName(user: UserInput!): User @lookup
}

input UserInput {
    info: Info
    name: String!
}

type Info {
    firstName: String!
}

interface User {
    name: String!
}

type EntraUser implements User {
    firstName: String!
}

type AdfsUser implements User {
    lastName: String!
}
```

</td>
<td>

🚨 There is no reasonable way to express this 

</td>

<td>

```graphql
@is(field: "profile: { firstName: <EntraUser>.firstname | <AdfsUser>.lastName} name")
```

</td>
<td>

```graphql
@is(field: "profile: { firstName: ... EntraUser { firstName } ... AdfsUser {lastName } } name")
```
</td>
</tr>

<tr>
<td>
Multiple Field Reference - Input field is deeper in the tree and output field is deeper in the tree
</td>
<td>

```graphql
type Query {
    findUserByName(user: UserInput!): User @lookup
}

input UserInput {
    info: Info
    name: String!
}

type Info {
    firstName: String!
}

type User {
    profile: Profile
    name: String!
}

type Profile {
    firstName: String!
}
```

</td>
<td>

🚨 There is no reasonable way to express this 

</td>

<td>

```graphql
@is(field: "info: { firstName: profie.firstName } name")
```

</td>
<td>

```graphql
@is(field: "info: { firstName: profile { firstName } } name")
```
</td>
</tr>

<tr>
<td>
Multiple Field Reference - Input field is deeper in the tree and output field is deeper in the tree with abstract type
</td>
<td>


```graphql
type Query {
    findUserByName(user: UserInput!): User @lookup
}

input UserInput {
    info: Info
    name: String!
}

type Info {
    firstName: String!
}

type User {
    profile: Profile
    name: String!
}

union Profile = EntraProfile | AdfsProfile

type EntraProfile {
    firstName: String!
}

type AdfsProfile {
    name: String!
}
```

</td>
<td>

🚨 There is no reasonable way to express this 

</td>

<td>

```graphql
@is(field: "info: { firstName: profile<EntraProfile>.firstName | profile<AdfsProfile>.name } name")
```

</td>
<td>

```graphql
@is(field: "info: { firstName: profile { ... EntraProfile { firstName } ... AdfsProfile { name } } } name")
```
</td>
</tr>

</table>