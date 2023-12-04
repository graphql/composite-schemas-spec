# Overview

GraphQL Fusion is a specification that describes how multiple GraphQL services,
known as _subgraphs_, are combined into a single unified GraphQL schema.

For clients querying the unified GraphQL schema the implementation details and complexities of
the distributed systems behind it are hidden and the observable behavior of the
distributed GraphQL executor is the same as of a standard GraphQL executor descibed
by the GraphQL specification.

GraphQL Fusion focuses on two components to allow interoperability between tolling and gatways.

- **Composition**: The schema composition describes a process of merging subgraph configurations
  into a single GraphQL schema that is annotated with execution directives and is referd to as the
  Gateway Configuration.

- **Execution**: The distributed GraphQL executor describes the Gateway Configuration and
  the core execution algorithms.