The GraphQL Composite Schemas spec describes a collaborative approach towards
build a single schema composed from multiple _source schemas_ by specifying the
algorithms to merge different GraphQL _subgraph_ schemas into a single
_composite schema_.

Two _source schema_ exposing a type with the same name form a _composite type_
in the _composite schema_.

```graphql example
# source schema 1
type SomeType {
  a: String
  b: String
}

# source schema 2
type SomeType {
  a: String
  c: String
}

# composite schema
type SomeType {
  a: String
  b: String
  c: String
}
```

## Subgraph

The GraphQL Composite Schemas spec refers to downstream GraphQL APIs that have
been designed for composition as subgraphs. These subgraphs may have additional
directives specified in the composition section to specify semantics of type
system members to the composition.

## Composite Schema

The result of a successful composition is a single GraphQL schema that is
annotated with execution directives. This schema document represents the
configuration for the distributed GraphQL executor and is called _composite
schema_.

## Entities

Entities are objects with a stable identity that endure over time. They
typically represent core domain objects that act as entry points to a graph. In
a distributed architecture, each _subgraph_ can contribute different fields to
the same entity, and is responsible for resolving only the fields that it
contributes. In such an architecture, entities effectively act as hubs that
enable transparent traversal across service boundaries.

## Keys

A representation of an object’s identity is known as a key. Keys consist of one
or more fields from an object, and are defined through the `@key` directive on
an object or interface type.

In a distributed architecture, it is unrealistic to expect all participating
systems to agree on a common way of identifying a particular type of entity. The
composite schemas spec therefore allows multiple keys to be defined for each
entity type, and each subgraph defines the particular keys that it is able to
support.

Lookups can also be nested if they can be reached through other lookups.

```graphql example
type Query {
  organization(id: ID!): Ogranization @lookup
}

type Ogranization {
  repository(name: String!): Repository @lookup
}

type Repository @key(fields: "id organization { id }") {
  name: String!
  organization: Ogranization
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