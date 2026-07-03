# Executor

A distributed GraphQL executor acts as an orchestrator that uses schema metadata
to rewrite a GraphQL request into a query plan. This plan resolves the required
data from subgraphs and coerces this data into the result of the GraphQL
request.

## Configuration

The supergraph is a GraphQL IDL document that contains metadata for the query
planner that describes the relationship between type system members and the type
system members on subgraphs.

## Resolving Default Field Implementations

A type annotated with `@partial` contributes default field implementations to
its target interface (see [@partial](#sec--partial)). This section defines how
the _distributed GraphQL executor_ resolves default fields at runtime.

**Fetching Default Fields**

To resolve the default fields of an entity whose concrete type implements the
target interface, the executor invokes the contributing source schema's partial
lookup with the interface's key values taken from the entity. The request
selects only default fields. It never includes `__typename` and never includes
inline fragments or type conditions: the contributing source schema does not
know the concrete types of the composite schema.

**Required Arguments**

Arguments of default fields that are annotated with `@require` are resolved by
the executor against the entity, following the standard `@require` runtime
behavior (see [@require](#sec--require) and
[Value Production](#sec-Value-Production)). The required data is fetched from
the source schemas that own the referenced fields before or together with the
partial fetch.

**Opaque Results**

A result produced by a partial-contributing source schema for an interface-typed
field is opaque: its concrete type is unknown. The executor may resolve further
fields of the interface contract through any source schema's lookup that returns
the interface, without knowing the concrete type. If the selection requires the
concrete type - `__typename`, type conditions, or fields that are not declared
on the interface - the executor must first resolve the concrete type through a
lookup field returning the interface (see
[Default Typename Unresolvable](#sec-Default-Typename-Unresolvable)); lookup
fields returning a partial type never provide concrete typing. A `__typename`
value obtained through a partial type's lookup is never surfaced to clients.

**Precedence Routing**

If an implementing type provides its own implementation of a defaulted field,
the executor routes that field to the source schema that owns the
implementation, not to the partial-contributing source schema. For all other
implementing types, the field is routed to the partial-contributing source
schema. Both routes may occur within a single selection set when the runtime
objects have mixed concrete types.

For example, given the interface `Media` with the default field `averageRating`
contributed by the partial type `MediaReviews`, and the implementing types
`Book` (no own implementation) and `Photo` (own implementation): a selection of
`averageRating` over a list of mixed `Media` objects routes `Book` objects to
the partial-contributing schema and `Photo` objects to the source schema that
defines `Photo.averageRating`.
