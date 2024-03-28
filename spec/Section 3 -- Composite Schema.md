# Composite Schema

The supergraph API schema is a subset of the supergraph schema that includes
just those types and fields that should be accessible by clients. Metadata used
by the router is excluded from the API schema by default. In addition, subgraphs
can mark definitions as @inaccessible to exclude them from the API schema. This
ensures these fields can only be used by the router (to access a key field and
use it as input to another subgraph for example) or by other subgraphs (by
requesting it in a @requires).
