# Overview

The GraphQL Composite Schemas specification describes how to construct a single
unified GraphQL schema, the _composite schema_, from multiple GraphQL schemas,
each termed a _source schema_.

The _composite schema_ presents itself as a regular _GraphQL schema_; the
implementation details and complexities of the underlying distributed systems
are not visible to clients, all observable behavior is the same as described by
the _GraphQL specification_.

The GraphQL Composite Schemas specification has a number of design principles:

- **Composable**: Rather than defining each _source schema_ in isolation and
  reshaping it to fit the _composite schema_ later, this specification
  encourages developers to design the source schemas as part of a larger whole
  from the start. Each source schema defines the types and fields it is
  responsible for serving within the context of the larger schema, referencing
  and extending what is provided by other source schemas. The GraphQL Composite
  Schemas specification does not describe how to combine arbitrary schemas.

- **Collaborative**: The GraphQL Composite Schemas specification is explicitly
  designed around team collaboration. By building on a principled composition
  model, it ensures that conflicts and inconsistencies are surfaced early and
  can be resolved before deployment. This allows many teams to contribute to a
  single schema without the danger of breaking it. The GraphQL Composite Schemas
  specification facilitates the coordinated effort of combining collaboratively
  designed source schemas into a single coherent composite schema.

- **Evolvable**: A _composite schema_ enables offering an integrated,
  product-centric API interface to clients. Each source _GraphQL service_ that
  underpins the composite schema should be able to evolve without disrupting
  these clients. Over time, the same functionality may be provided by a
  different combination of services, while the composite schemas interface should
  continue to support existing requests from all clients; source schema
  boundaries are therefore considered an implementation detail and should not
  be exposed to clients.

- **Explicitness**: To make the composition process easier to understand and to
  avoid ambiguities that can lead to confusing failures as the system grows, the
  GraphQL Composite Schemas specification prefers to be explicit about
  intentions and minimize reliance on inference and convention.

To enable greater interoperability between different implementations of tooling
and gateways, this specification focuses on two core components: schema
composition and distributed execution.

- **Schema Composition**: Schema composition describes the process of merging
  multiple _source schemas_ into a single GraphQL schema, the _composite
  schema_. During this process, an intermediary schema, the _composite execution
  schema_, is generated. This composite execution schema is annotated with
  directives to describe execution, and may have additional internal fields that
  won't be exposed in the client-facing _composite schema_.

- **Distributed Execution**: The _distributed GraphQL executor_ specifies the
  core execution behavior and algorithms that enable fulfillment of a _GraphQL
  request_ performed against the _composite schema_.
