# RFC: Handling Merging of Enums

## Abstract

In federated GraphQL architectures, merging enums defined across different subgraphs presents challenges if the enums are used in both input and output types. Inconsistent enum definitions can lead to runtime errors, composition errors, and hinder schema evolvability. This document explores the problems arising from enum merging, providing concrete examples, and proposes solutions to handle enum merging effectively. 

## Background

In a federated schema, multiple subgraphs contribute to a unified API presented by the gateway. Enums are scalar types that define a set of possible values. When subgraphs define enums with the same name but different values, these enums must be merged at the gateway level. There are different strategies for merging enums, each with its trade-offs and implications.

### Example Scenario: Order Processing System

Consider an e-commerce platform with an order processing system where different subgraphs handle various aspects of order management:

- **Subgraph A**: Manages order creation and standard shipping methods.
- **Subgraph B**: Manages express and international shipping options.

Each subgraph defines an enum `ShippingMethod` with values relevant to its domain.

**Subgraph A**

```graphql
enum ShippingMethod {
  STANDARD
  ECONOMY
}
```

**Subgraph B**

```graphql
enum ShippingMethod {
  EXPRESS
  INTERNATIONAL
}
```

At the gateway, naive merging of these enums would result in:

**Gateway**

```graphql
enum ShippingMethod {
  STANDARD
  ECONOMY
  EXPRESS
  INTERNATIONAL
}
```


### Open Questions

Before deciding on an approach for merging enums, it's important to consider several fundamental questions:

1. **Should Enums Be Consistent Across Subgraphs?**

   - **Consistency Expectation**: Do we expect enums to be consistent across all subgraphs, or can they differ?
   - **Implications**: If enums must be the same across subgraphs (e.g., shared types), all options are viable. If enums differ across subgraphs (e.g., `UserKind` enums that are not aligned), some methods may not work, and only context-based merging may be applicable.

2. **Common Usage of Enums: Single-Direction or Bi-Directional?**

   - **Usage Patterns**: Are shared enums most commonly used only as input, only as output, or both?
   - **Workflow Optimization**: If most shared enums are only input or output, it's unwise to choose an approach that requires a lot of ceremony to implement changes. Optimizations in workflow depend on whether we optimize for single-direction or bi-directional enums.

3. **Team Velocity and Coordination**

   - **Independent Evolution**: In enterprises, teams often want to move at different speeds. Some approaches support this better than others.
   - **Blocking Issues**: With strict equality, a team may be blocked until other subgraphs are updated and deployed, affecting development velocity.

4. **Schema Governance Policies**

   - **Input and Output Separation**: Organizations may enforce policies that no enum is ever used as both input and output (e.g., using `UserStatusInput` and `UserStatus`).
   - **Impact on Evolution**: This separation can simplify evolution and remove the need for strict equality, as changes in output enums don't affect input enums.

5. **Centralized Type Repository**

   - **Domain-Wide Consistency**: Ensuring domain-wide enum equality can be achieved through a central type repository where all enums are stored.
   - **Deployment Coordination**: Updating enums requires all subgraphs to redeploy with the updated enums, which can be managed through schema governance.

6. **Balancing Runtime and Composition Errors**

   - **Error Timing**: It's important to alert developers as early as possible to issues, preferably during composition rather than at runtime.
   - **Schema Evolution**: However, we should not completely block schema evolution during composition. A balance must be struck to allow for progress while maintaining safety.

## Problem Statement

Merging enums across subgraphs can lead to several issues that affect both the composition of the schema and its runtime behavior. The main problems are:

1. **Runtime Errors Due to Unrecognized Enum Values**
2. **Errors Not Surfacing Early**
3. **Difficulty in Schema Evolution**

### 1. Runtime Errors Due to Unrecognized Enum Values

Inconsistent enum definitions can cause runtime errors when:

- **Input Enums**: A client sends an enum value in a mutation that is not recognized by the subgraph handling the request.
- **Output Enums**: A subgraph returns an enum value not defined in the gateway's schema, causing serialization errors at the gateway.

**Examples:**

- **Input Error**: A client attempts to create an order with `shippingMethod: EXPRESS`, but Subgraph A, which handles order creation, does not recognize `EXPRESS`, leading to a runtime error in the subgraph.

  ```graphql
  mutation {
    createOrder(input: { shippingMethod: EXPRESS }) {
      order {
        id
        shippingMethod
      }
    }
  }
  ```

- **Output Error**: Subgraph B returns an order with `shippingMethod: EXPRESS`, but if the gateway's enum does not include `EXPRESS`, the gateway cannot serialize the response, resulting in a runtime error.

### 2. Errors Not Surfacing Early

When there are no composition-time errors, developers may not be aware of inconsistencies until they manifest as runtime failures in production. Preferably, errors should be caught during schema composition to provide immediate feedback to developers.

### 3. Difficulty in Schema Evolution

Exposing a superset or intersection makes evolving the schema challenging:

- **Removing Values**: Removing an enum value is complicated if subgraphs still rely on it.
- **Adding Values**: Introducing a new enum value requires coordination among all subgraphs to prevent inconsistencies.

## Proposed Solutions

To address these problems, we consider four options for handling enum merging. Each option is analyzed with integrated pros, cons, and examples.

### Option 1: Strict Equality with Inaccessible Values

#### Approach

- **Enforce Strict Equality**: Enum definitions must be identical across subgraphs.
- **Use `@inaccessible` Directive**: Subgraph-specific enum values are annotated with `@inaccessible` to hide them from the gateway schema.

#### Implementation

**Subgraph A**

```graphql
enum ShippingMethod {
  STANDARD
  ECONOMY
  INTERNATIONAL @inaccessible
}
```

**Subgraph B**

```graphql
enum ShippingMethod {
  STANDARD
  ECONOMY
  EXPRESS @inaccessible
  INTERNATIONAL
}
```

**Gateway**

```graphql
enum ShippingMethod {
  STANDARD
  ECONOMY
}
```

#### Pros and Cons

**Pros:**

- **Prevents Runtime Errors**: By ensuring that only recognized enum values are exposed, runtime errors are minimized.
- **Composition-Time Validation**: Differences in enum definitions are caught during schema composition.
- **Explicit Control**: Developers explicitly manage enum exposure, increasing schema clarity, with the hope that they will not return the inaccessible values until they are exposed.
- **Supports Evolvability**: A new enum value can be added by flagging it as `@inaccessible` in a subgraph, which hides it from the gateway schema. Other subgraphs can then expose it when ready. The flag can be removed once all subgraphs are updated.

**Cons:**

- **Annotation Overhead**: Requires developers to use `@inaccessible`.
- **Runtime Errors Still Possible**: If a subgraph returns an inaccessible value, it may cause serialization errors.

### Option 2: Contextual Merging Based on Usage

#### Approach

- **Enum Used Only as Output**: Use the union (superset) of all enum values from subgraphs.
- **Enum Used Only as Input**: Use the intersection of enum values common to all subgraphs.
- **Enum Used in Both Input and Output**: Enforce strict equality across subgraphs.

#### Implementation

Assuming `ShippingMethod` is used both as input and output:

- **Gateway Enforces Strict Equality**: Enum definitions must be identical.

If an enum is only used as an output or input:

- **Output-Only Enum (e.g., `OrderStatus`)**

  **Subgraph A**

  ```graphql
  enum OrderStatus {
    PENDING
    SHIPPED
  }
  ```

  **Subgraph B**

  ```graphql
  enum OrderStatus {
    DELIVERED
    RETURNED
  }
  ```

  **Gateway**

  ```graphql
  enum OrderStatus {
    PENDING
    SHIPPED
    DELIVERED
    RETURNED
  }
  ```

- **Input-Only Enum (e.g., `CancellationReason`)**

  **Subgraph A**

  ```graphql
  enum CancellationReason {
    OUT_OF_STOCK
    CUSTOMER_REQUEST
  }
  ```

  **Subgraph B**

  ```graphql
  enum CancellationReason {
    FRAUD_DETECTED
    PAYMENT_ISSUE
  }
  ```

  **Gateway**

  ```graphql
  enum CancellationReason {
    // Intersection is empty; leads to a composition error
  }
  ```

#### Pros and Cons

While this approach is pretty similar to the strict equality approach when enums are used in both input and output, it is much easier to manage when enums are used only as input or output.

**Pros:**

- **Maximizes Output Information**: Clients receive comprehensive data in outputs.
- **Minimizes Input Errors**: Clients can only use input values recognized by all subgraphs.

**Cons:**

- **Complexity**: Understanding different merging strategies increases complexity.
- **Limited Evolvability for Enums Used in Both Contexts**: Requires strict equality, which may hinder independent evolution.
- **Limited Evolvability for Enums Changing Context**: If an enum transitions from output-only to shared, it requires a change in merging strategy and changes in all subgraphs.

### Option 3: Intersection Merging

#### Approach

- **Merge by Intersection**: Combine enums by including only the values common to all subgraphs.
- **No Annotations**: Developers do not use explicit directives to manage enum exposure.
- **Uniform Enum Definitions**: Enums used in both input and output types at the gateway contain only the intersecting values.

#### Implementation

**Subgraph A**

```graphql
enum ShippingMethod {
  STANDARD
  ECONOMY
  EXPRESS
}
```

**Subgraph B**

```graphql
enum ShippingMethod {
  ECONOMY
  EXPRESS
  INTERNATIONAL
}
```

**Gateway**

```graphql
enum ShippingMethod {
  ECONOMY
  EXPRESS
}
```

#### Pros and Cons

**Pros:**

- **Simplified Composition**: No need for developers to use `@inaccessible`; merging is straightforward based on common values.
- **Reduced Composition Errors**: Discrepancies are minimized during schema composition.

**Cons:**

- **Developer Unawareness**: Without explicit directives, developers might not realize that certain enum values are omitted from the gateway schema until runtime errors occur.
- **Runtime Errors Potential**: If a subgraph returns an enum value not present in the gateway's enum, it leads to serialization errors at runtime.
- **Evolvability Constraints**: Adding new enum values requires coordination across all subgraphs to ensure inclusion in the intersection, hindering independent evolution.

### Option 4: Contextual Merging with Directives

#### Approach

- **Allow Developers to Force Enums to Be Input-Only or Output-Only**: Use directives (e.g., `@input` and `@output`) to specify the intended use of enums.
- **Merge Based on Usage**: Enums annotated as input-only or output-only are merged accordingly.

#### Implementation

- **For Enums Used Only as Output**

  **Subgraph A**

  ```graphql
  enum OrderStatus @output {
    PENDING
    SHIPPED
  }
  ```

  **Subgraph B**

  ```graphql
  enum OrderStatus @output {
    DELIVERED
    RETURNED
  }
  ```

  **Gateway**

  ```graphql
  enum OrderStatus {
    PENDING
    SHIPPED
    DELIVERED
    RETURNED
  }
  ```

- **For Enums Used Only as Input**

  **Subgraph A**

  ```graphql
  enum CancellationReason @input {
    OUT_OF_STOCK
    CUSTOMER_REQUEST
  }
  ```

  **Subgraph B**

  ```graphql
  enum CancellationReason @input {
    FRAUD_DETECTED
    PAYMENT_ISSUE
  }
  ```

  **Gateway**

  ```graphql
  enum CancellationReason {
    // Intersection of input enums; may result in an empty enum or require special handling
  }
  ```

- For Enums Used in Both Contexts

  **Subgraph A**

  ```graphql
  enum ShippingMethod {
    STANDARD @inaccessible
    ECONOMY
    EXPRESS
  }
  ```

  **Subgraph B**

  ```graphql
  enum ShippingMethod {
    ECONOMY
    EXPRESS
    INTERNATIONAL @inaccessible
  }
  ```

  **Gateway**

  ```graphql
  enum ShippingMethod {
    ECONOMY
    EXPRESS
  }
  ```

#### Pros and Cons

**Pros:**

- **Developer Control**: Developers can explicitly specify how enums should be used, providing greater flexibility.
- **Optimized Merging**: Enums are merged based on their specific usage, potentially reducing conflicts.
- **Easier Evolution**: Teams can add new enum values without affecting other teams if the enums are used in a single direction.

**Cons:**

- **Increased Complexity**: Introducing new directives adds complexity to the schema and requires developers to learn and apply them correctly.
- **Runtime Errors**: If a subgraph returns an enum value not present in the gateway's enum, it leads to serialization errors at runtime.
- **Still Requires Strict Equality for Shared Enums**: Enums used in both input and output types must still be strictly equal across subgraphs.
- **Schema Evolution Challenges**: Transitioning an enum from input-only to shared requires changing the directive and updating all subgraphs, which can be cumbersome.
