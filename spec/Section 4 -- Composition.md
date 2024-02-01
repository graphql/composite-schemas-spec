# Composition

Schema composition describes the process of merging multiple _source schemas_
into a single GraphQL schema, the _composite schema_. During this process, an
intermediary schema, the _composite execution schema_, is generated. This
composite execution schema is annotated with directives to describe execution,
and may have additional internal fields that won't be exposed in the
client-facing _composite schema_. The composition process is divided into four
major algorithms, `Validate Source Schemas`, `Merge Source Schemas` and
`Validate Satisfiability`, `Create Composite Schema` that are run in order.

## Validate Source Schemas

Validat GraphQL Document / Valdate Directives

## Merge Source Schemas

Validate if can be merged / Apply Merge -> _composite execution schema_

## Validate Satisfiability

Validate if all query pathes are can be resolved

## Create Composite Schema

transform _composite execution schema_ -> _composite schema_
