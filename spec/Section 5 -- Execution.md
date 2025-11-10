# Executor

A distributed GraphQL executor acts as an orchestrator that uses schema metadata
to rewrite a GraphQL request into a query plan. This plan resolves the required
data from subgraphs and coerces this data into the result of the GraphQL
request.

## Configuration

The supergraph is a GraphQL IDL document that contains metadata for the query
planner that describes the relationship between type system members and the type
system members on subgraphs.

## PlanOptions Algorithm

The PlanOptions algorithm computes which source schemas (options) can resolve a specific path in the
composite schema. The algorithm examines each step of the path to determine which source schemas can
fulfill the requested fields.

### Formal Specification

me 

Query.me.profile.age
[
  (Query, me),
  (User, profile),
  (Profile, age)
]

Mutation.createUser.query.me 
[
  (Mutation, createUser),
  (CreateUserPayload, query),
  (Query, me)
]
// Technically the first item doesnt have to check all source schemas twice (inintializ eoptions as [])
//



PlanOptions(path):
- Let {pathElements} be the list of tuples ({type}, {field}) in the provided {path}.

- Let ({initialType}, {initialField}) be the first element in {pathElements}.

- Let {sourceSchemas} be an empty set.
- For each {sourceSchema} in all source schemas:
  - If {sourceSchema} does not define {initialType}.{initialField}
    - Continue to the next {sourceSchema}.
  - Add {sourceSchema} to {sourceSchemas}.

PlanOptionsInternal(path, sourceSchemas):
- For each {pathElement} in {pathElements} starting from the second element:
  - Initialize a new empty set {nextSchemas}.
  - Set {currentType} and {currentField} to the respective elements of the current {pathElement}.
  - For each {currentSchema} in {sourceSchemas}:
    - For each {candidateSchema} in all source schemas:
      - If {candidateSchema} does not define {currentType}.{currentField}
        - Continue to the next {candidateSchema}.

      - If {candidateSchema} not equals {currentSchema}: 
        - If {IsReachable(currentSchema, candidateSchema, currentType)} returns false:
            - Continue to the next {candidateSchema}.

      - If {currentField} on {currentType} in {candidateSchema} defines a requirement:
        - If {ResolveRequirement(currentSchema, candidateSchema, currentType, currentField)} returns false:
            - Continue to the next {candidateSchema}.

      - Add {candidateSchema} to {nextSchemas}.

  - If {nextSchemas} is empty:
    - Return an empty set.

  - Set {sourceSchemas} equal to {nextSchemas} to proceed to the next element in the path.

- return {sourceSchemas}.

IsReachable(sourceSchema, targetSchema, type):
- If {targetSchema} does not define {type}
  - return false
- If {sourceSchema} equals {targetSchema}
  - Assert: TODO This should be a precondition
- Let {lookups} be the set of all lookup fields in {targetSchema} that return {type}
- For each {lookup} in {lookups}:
  - Let {keyFields} be the set of fields that form the key for {lookup} in {targetSchema}
  - For each {keyField} in {keyFields}:
    - If {ResolveRequirement(sourceSchema, targetSchema, type, keyField)} returns false
      - // Continue to the next {lookup}
  - return true
- return false

ResolveRequirement(currentSchema, candidateSchema, type, field):
- Let {requirements} be the parsed selection map from requirements on {field}
- Let {otherSchemas} be the set of all source schemas excluding {candidateSchema} and {currentSchema}
- For each {requiredField} in {requirements}:
  - Let {fieldPath} be the path to {requiredField} starting from {type}
  - Let {canResolve} be false
  - If {currentSchema} defines {type} and {fieldPath}
    - If {PlanOptionsInternal(fieldPath, [currentSchema])} returns an non empty set
      - Set {canResolve} to true
  - For each {otherSchema} in {otherSchemas}:
    - If {otherSchema} defines {type} and {fieldPath}
      - // IsReachable
        - If {PlanOptionsInternal(fieldPath, [currentSchema])} returns an non empty set
          - Set {canResolve} to true
          - Break
  - If {canResolve} is false
    - return false
- return true

Notes:
{ user { profile { name id }} }
user.profile.name
user.profile.id
user.profile.id -> UserSerivce
profile.name -> ProfileService through a Lookup on user

