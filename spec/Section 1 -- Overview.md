# Overview

The GraphQL Composite Schemas specification describes how to construct a single unified GraphQL schema, the _composite schema_, from multiple GraphQL schemas, the _source schemas_.

The composite schema presents itself as a regular GraphQL schema; the implementation details and complexities of the underlying distributed systems are not visible to clients, all observable behavior is the same as described by the GraphQL specification.

GraphQL Composite Schemas specification has a number of design principles:

- **Composable**: Rather than defining GraphQL schemas in isolation and
  reshaping them to fit a composed graph later, it encourages developers to
  think of GraphQL schemas as partial schemas that are designed as part of a
  larger schema from the get go. Each source schema defines the types and fields
  it is responsible for serving within the context of a larger schema,
  referencing and extending what is already there. In other words, the GraphQL
  Composite Schemas specification is not designed to glue arbitrary schemas
  together, but to facilitate a coordinated effort to build a coherent composite
  schema.

- **Collaborative**: The GraphQL Composite Schemas specification is explicity
  designed around team collaboration. By building on a principled composition
  model, it ensures that conflicts and inconsistencies are surfaced early and
  can be resolved before deploying. This allows many teams to contribute to a
  single schema without the danger of breaking it.

- **Evolvable**: A composite schema offers an integrated, product-centric view
  of underlying services. And these services should be able to evolve without
  disrupting clients. Over time, the same functionality may be provided by a
  different combination of services, while the schema surface stays the same.
  Source schema boundaries are therefore considered an implementation detail and
  should never be exposed to clients.

- **Explicitness**: This specification values explicitness over inferring
  intentions, as these tend to break down and can lead to ambiguity over time.

This specification focuses on two core components to allow interoperability
between tooling and gateways from different implementers, the schema composition
and the execution.

- **Composition**: The schema composition describes the process of merging
  multiple source schemas into a single GraphQL schema that is annotated with
  execution directives and is referred to as the composite schema.

- **Execution**: The distributed GraphQL executor specifies the core execution
  behavior and algorithms.
