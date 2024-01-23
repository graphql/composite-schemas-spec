# Overview

The GraphQL Composite Schemas specification describes how multiple GraphQL
schemas, known as _source schemas_, are combined into a single unified GraphQL
schema called the _composite schema_.

For clients querying the GraphQL composite schema, the implementation details and
complexities of the underlying distributed systems are hidden. The observable
behavior of the distributed GraphQL executor is the same as that of a standard
GraphQL executor as described by the GraphQL specification.

This specification focuses on two core components to allow interoperability
between tooling and gateways from different implementers, the schema composition
and the execution.

- **Composition**: The schema composition describes a process of merging
  multiple source schemas into a single GraphQL schema that is annotated with
  execution directives and is referred to as the composite schema.

- **Execution**: The distributed GraphQL executor specifies the core execution behavior and algorithms.

The GraphQL Composite Schemas spec describes a collaborative approach towards
build a single schema composed from multiple _source schemas_ by specifying the
algorithms to merge different GraphQL _subgraph_ schemas into a single
_composite schema_.

Two _source schema_ exposing a type with the same name form a _composite type_ in the
_composite schema_.

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
configuration for the distributed GraphQL executor and is called _composite schema_.

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
