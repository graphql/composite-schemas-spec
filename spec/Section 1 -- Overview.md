# Overview

The GraphQL Composite Schemas specification describes how multiple GraphQL
schemas, known as _source schemas_, are combined into a single unified GraphQL
schema called the _composite schema_.

For clients querying the GraphQL composite schema, the implementation details
and complexities of the underlying distributed systems are hidden. The
observable behavior of the distributed GraphQL executor is the same as that of a
standard GraphQL executor as described by the GraphQL specification.

This specification focuses on two core components to allow interoperability
between tooling and gateways from different implementers, the schema composition
and the execution.

- **Composition**: The schema composition describes a process of merging
  multiple source schemas into a single GraphQL schema that is annotated with
  execution directives and is referred to as the composite schema.

- **Execution**: The distributed GraphQL executor specifies the core execution
  behavior and algorithms.