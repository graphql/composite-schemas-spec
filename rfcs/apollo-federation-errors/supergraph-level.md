# Supergraph-level Validation Errors

A list of validation errors (not all of them) that can be returned only when validating a
supergraph, with context from all subgraphs.

### TYPE_KIND_MISMATCH

```
Type "Product" has mismatched kind: it is defined as Object Type in subgraph Foo but Interface Type in subgraph Bar
```

### NO_QUERIES

```
No queries found in any subgraph: a supergraph must have a query root type.
```

### REQUIRED_INPUT_FIELD_MISSING_IN_SOME_SUBGRAPH

```
Input object field "AddProductInput.price" is required in some subgraphs but does not appear in all subgraphs: it is required in subgraphs Foo and Bar, but does not appear in Baz.
```

### REQUIRED_INACCESSIBLE

```
Input field "AddProductInput.price" is @inaccessible but is a required input field of its type.

Argument "Product.price(currency:)" is @inaccessible but is a required argument of its field.
```

### REQUIRED_ARGUMENT_MISSING_IN_SOME_SUBGRAPH

```
Argument "Product.price(currency:)" is required in some subgraphs but does not appear in all subgraphs: it is required in subgraphs Foo and Bar, but does not appear in Baz.
```

### REFERENCED_INACCESSIBLE

```
Type "Price" is @inaccessible but is referenced by "Product.price", which is in the API schema.

Type "CurrencyInput" is @inaccessible but is referenced by "Product.price(currency:)", which is in the API schema.
```

### ONLY_INACCESSIBLE_CHILDREN

```
Type "CategoryEnum" is in the API schema but all of its values are @inaccessible.

Type "ProductObject" is in the API schema but all of its fields are @inaccessible.
Type "ProductInterface" is in the API schema but all of its fields are @inaccessible.
Type "AddProductInput" is in the API schema but all of its fields are @inaccessible.
```

### INVALID_FIELD_SHARING

```
Fields on root level subscription object cannot be marked as shareable.

Non-shareable field "Product.price" is resolved from multiple subgraphs: it is resolved from subgraphs Foo and Bar, and defined as non-shareable in Bar.

Non-shareable field "Product.price" is resolved from multiple subgraphs: it is resolved from subgraphs Foo and Bar, and defined as non-shareable in all of them.
```

### INTERFACE_KEY_MISSING_IMPLEMENTATION_TYPE

```
[Bar] Interface type "Node" has a resolvable key (@key(fields: "<value>")) in subgraph "Bar" but that subgraph is missing some of the supergraph implementation types of "Node". Subgraph "Bar" should define types "Product" and "Price" (and have them implement "Node").
```

### INTERFACE_FIELD_NO_IMPLEM

```
Interface field "ProductIf.name" is declared in subgraphs Foo and Bar but type "Book", which implements "ProductIf" in subgraph "Baz" does not have field "name".
```

### EMPTY_MERGED_INPUT_TYPE

```
None of the fields of input object type "AddProductInput" are consistently defined in all the subgraphs defining that type. As only fields common to all subgraphs are merged, this would result in an empty type.
```

#### INPUT_FIELD_DEFAULT_MISMATCH

```
Input field "AddProductInput.isAvailable" has incompatible default values across subgraphs: it has default value "true" in subgraph Foo, but default value "false" in subgraph Bar.
```

### FIELD_TYPE_MISMATCH

Applies to input object types, object types, and interface types.

```
Type of field "Product.price" is incompatible across subgraphs: it has type "Price" in subgraph Foo, but type "Float" in subgraph Bar.
```

### FIELD_ARGUMENT_TYPE_MISMATCH

```
Type of argument "Product.price(currency:)" is incompatible across subgraphs: it has type "Currency"
in subgraph Foo, but type "String" in subgraph Bar.
```

### FIELD_ARGUMENT_DEFAULT_MISMATCH

```
Argument "Product.price(currency:)" has incompatible default values across subgraphs: it has default value "USD" in subgraph Foo, but default value "EUR" in subgraph Bar.
```

### EXTERNAL_TYPE_MISMATCH

```
Type of field "Product.price" is incompatible across subgraphs (where marked @external): it has type "Price" in subgraph Foo, but type "Float" in subgraph Bar.
```

### EXTERNAL_MISSING_ON_BASE

```
Type "Product" is marked @external on all the subgraphs in which it is listed (Foo, Bar).

Field "Product.price" is marked @external on all the subgraphs in which it is listed (Foo, Bar).
```

### EXTERNAL_ARGUMENT_MISSING

```
Field "Product.price" is missing argument "currency" in some subgraphs where it is marked @external: argument "currency" is declared in subgraphs Foo and Bar, but not in subgraph Baz (where "Product.price" is @external)
```

### EXTENSION_WITH_NO_BASE

```
[Bar] Type "Product" is an extension type, but there is no type definition for "Product" in any subgraph.
```

### ENUM_VALUE_MISMATCH

```
Enum type "CurrencyEnum" is used as both input type (for example, as type of "AddProductInput.currency") and output type (for example, as type of "Product.price.currency"), but value "PLN" is not defined in all the subgraphs defining "CurrencyEnum": "PLN" is defined in subgraph Foo, but not in subgraphs Bar and Baz.
```

### EMPTY_MERGED_ENUM_TYPE

```
None of the values of enum type "Currency" are defined consistently in all the subgraphs defining that type. As only values common to all subgraphs are merged, this would result in an empty type.
```

### DEFAULT_VALUE_USES_INACCESSIBLE

```
Enum value "Currency.PLN" is @inaccessible but is used in the default value of "AddProductInput.price", which is in the API schema.
```

### SATISFIABILITY_ERROR

Next to every error message, there's an example query path that cannot be satisfied by the
supergraph.

```
Cannot move to subgraph "foo" using @key(fields: "<value>") of "Product", the key field(s) cannot be resolved from subgraph "bar".

Cannot satisfy @require conditions on field "Product.price".

Field "Product.price" is not resolvable because marked @external.

Cannot find field "Product.prince".

Cannot move to subgraph "bar", which has field "Product.name", because type "Product" has no @key defined in subgraph "foo".


```
