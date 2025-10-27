# Subgraph-level Validation Errors

A list of validation errors (not all of them) that can be returned when validating a subgraph, in
isolation.

### QUERY_ROOT_TYPE_INACCESSIBLE

The root query type must be accessible.

Example error message:

```
Type "Query" is @inaccessible but is the root query type, which must be in the API schema.
```

### INVALID_SHAREABLE_USAGE

Example error message:

```
Invalid use of @shareable on field "Product.name": only object type fields can be marked with @shareable
```

### REQUIRES_UNSUPPORTED_ON_INTERFACE

Example error messages:

```
Cannot use @requires on field "Product.price" of parent type "Product": @requires is not yet supported within interfaces.
```

### REQUIRES_INVALID_FIELDS_TYPE

```
On field "Product.price", for @requires(fields: "<value>"): Invalid value for argument "fields": must be a string.
```

### REQUIRES_INVALID_FIELDS

```
On field "Product.price", for @requires(fields: "<value>"): <error message>

On field "Product.price", for @requires(fields: "<value>"): Cannot query field "uuid" on type "Product" (if the field is defined in another subgraph, you need to add it to this subgraph with @external).

On field "Product.price", for @requires(fields: "<value>"): Unknown directive "@<directive-name>" in selection
```

### REQUIRES_DIRECTIVE_IN_FIELDS_ARG

```
On field "Product.price", for @requires(fields: "<value>"): cannot have directive applications in the @requires(fields:) argument but found @<directive-name>.
```

### REQUIRES_FIELDS_MISSING_EXTERNAL

```
On field "Product.price", for @requires(fields: "<value>"): field "Product.uuid" should not be part of a @requires since it is already provided by this subgraph (it is not marked @external)
```

### PROVIDES_ON_NON_OBJECT_FIELD

```
Invalid @provides directive on field "Price.price": field has type "Float!" which is not a Composite Type
```

### PROVIDES_UNSUPPORTED_ON_INTERFACE

```
Cannot use @provides on field "Product.price" of parent type "Product": @provides is not yet supported within interfaces
```

### PROVIDES_INVALID_FIELDS_TYPE

```
On field "Product.price", for @provides(fields: "<value>"): Invalid value for argument "fields": must be a string.
```

### PROVIDES_INVALID_FIELDS

```
On field "Product.price", for @provides(fields: "<value>"): <error message>

On field "Product.price", for @provides(fields: "<value>"): Invalid empty selection set for field "Price.details" of non-leaf type "Price"

On field "Product.price", for @provides(fields: "<value>"): Cannot query field "details" on type "Price" (if the field is defined in another subgraph, you need to add it to this subgraph with @external).

On field "Product.price", for @provides(fields: "<value>"): Unknown directive "@<directive-name>" in selection
```

### PROVIDES_DIRECTIVE_IN_FIELDS_ARG

```
On field "Product.price", for @provides(fields: "<value>"): cannot have directive applications in the @provides(fields:) argument but found @<directive-name>.
```

### PROVIDES_FIELDS_HAS_ARGS

```
On field "Product.price", for @provides(fields: "<value>"): field "Price.details" cannot be included because it has arguments (fields with argument are not allowed in @provides)
```

### PROVIDES_FIELDS_MISSING_EXTERNAL

```
On field "Product.price", for @provides(fields: "<value>"): field "Price.details" should not be part of a @provides since it is already provided by this subgraph (it is not marked @external)

On field "Product.price", for @provides(fields: "<value>"): field "Price.details" should not be part of a @provides since it is already "effectively" provided by this subgraph (while it is marked @external, it is a @key field of an extension type, which are not internally considered external for historical/backward compatibility reasons)
```

### KEY_UNSUPPORTED_ON_INTERFACE

```
Cannot use @key on interface "Node": @key is not yet supported on interfaces
```

### KEY_INVALID_FIELDS_TYPE

```
On type "Product", for @key(fields: "<value>"): Invalid value for argument "fields": must be a string.
```

### KEY_INVALID_FIELDS

```
On type "Product", for @key(fields: "<value>"): <error-message>

On type "Product", for @key(fields: "<value>"): Cannot query field "name" on type "Product" (the field should either be added to this subgraph or, if it should not be resolved by this subgraph, you need to add it to this subgraph with @external).

On type "Product", for @key(fields: "<value>"): Unknown directive "@<directive-name>"
```

### KEY_DIRECTIVE_IN_FIELDS_ARG

```
On type "Product", for @key(fields: "<value>"): cannot have directive applications in the @key(fields:) argument but found @<directive-name>.
```

### KEY_FIELDS_HAS_ARGS

```
On type "Product", for @key(fields: "<value>"): field Product.name cannot be included because it has arguments (fields with argument are not allowed in @key)
```

### KEY_FIELDS_SELECT_INVALID_TYPE

```
On type "Product", for @key(fields: "<value>"): field "Product.price" is a Interface type which is not allowed in @key
```

### INTERFACE_KEY_NOT_ON_IMPLEMENTATION

```
Key @key(fields: "<value>") on interface type "Product" is missing on implementation type "<type-name>".
```

### MERGED_DIRECTIVE_APPLICATION_ON_EXTERNAL

It applies mostly to `@tag` and `@inaccessible` directives. They cannot overlap with `@external`
fields.

```
Cannot apply merged directive <directive> to external field "Product.price"
```

### EXTERNAL_UNUSED

```
Field "Product.name" is marked @external but is not used in any federation directive (@key, @provides, @requires) or to satisfy an interface; the field declaration has no use and should be removed (or the field should not be @external).
```
