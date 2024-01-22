# GraphQL Composite Schemas Specification Roadmap

## Mission

_Provide a specification that allows to build GraphQL Gateways and GraphQL
composition tooling with different implementations and technology stacks to
interact freely if GraphQL composition tooling and GraphQL gateways are
compliant._

## Guiding principles

- Development is based on use cases
- Strive for backwards-compatible progress

## Version 1.0

Version 1 aims to codify a minimal core specification that specifies a set of
subgraph and supergraph directives. Further, it aims to codify the core
composition and execution algorithms for GraphQL composition tooling and GraphQL
gateways.

In layout and structure version 1.0 should lay a foundation for future
development and standardization.

### Scope

- Subgraph Directives
- Supergraph Directives
- Schema Composition
- Distributed Executor

## Stages

The process of writing this specification may proceed according this rough
outline of stages. We are currently in the _Proposal Stage_.

### Stage 0: Preliminary

In the _Preliminary Stage_, things may change rapidly and no one should count on
any particular detail remaining the same.

- If a PR has no requests for changes for 2 weeks then it should be merged by
  one of the maintainers
- If anyone has an objection later, they just open a PR to make the change and
  it goes through the same process
- Optional: When there is lots of consensus but not 100% full consensus then:
  - We might merge the consensus-view and debate modifying it in parallel
  - Anyone can extract the non-controversial part and make a separate PR

When the spec seems stable enough, the working group would promote it to
_Proposal Stage_.

### Stage 1: Proposal

In the _Proposal Stage_, things can still change but it is reasonable to start
implementations.

- Before release of the spec, in "Draft" stage, we have to review the spec and
  review all open PRs
- Every merge to master would need strong consensus
- Only changes that address concerns
- Implementers could start trying things

After the spec and open PRs are reviewed and there is strong consensus, the
working group would promote it to _Draft Stage_.

### Stage 2: Draft

This corresponds to the general
[GraphQL Draft Stage](https://github.com/graphql/graphql-spec/blob/master/CONTRIBUTING.md#stage-2-draft)
