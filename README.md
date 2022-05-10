# Iris spec

An alternative type system for GraphQL

## Motivation

<img alt="Logo" align="right" src="https://github.com/iris-qraphql/iris-spec/img/logo.svg" width="15%" />

The motivation of Iris is to combine the flexibility of the GraphQL query language with the formalism and strength of the Haskell type system.

iris addresses the following issues related to GraphQL:

- GraphQL forces the client to select fields of types, which is not well suited for server-to-server applications (where you don't need selectivity, like in RPC) and is causing difficulties with recursive data types (like [tree types](https://github.com/iris-qraphql/iris-spec/blob/main/why-gql-needs-typed-scalars.md)).
- Default values for GraphQL inputs are dangerous for recursive data types.
- GraphQL does not support input unions
- the separation between input and output types and the use of multiple entities to cover sum and product types is not always practical and leads to a verbose schema definition.

Iris solutions.

- in Iris you can use "data" types that can be used for both input and output and do not force the client to select fields.
- default values in iris are only allowed on arguments.
- iris uses only 3 entities `wrapping`, `data` and `resolver` types. where `data` and `resolver` are represented with identical `ADT` syntax. the type `data` covers also the case of input union.

the Language attempts to substitute various entities of the GQL language (such as input, scalar, enum, object, enum, interface, and wrapping types) with small but more unified and powerful alternatives (such as `resolver`, `data`, and `wrapping` types).

The types defined by Iris can be converted into the standard GQL language that can be used by GraphQL clients. However, these converted types have additional annotations that provide additional information (like JSDoc) that can be used by code genes to generate suitable types for them.

## Type System

Iris type systems based on two type categories: ADTs and wrapping types. These types generalize GraphQL scalars, enums, inputObject, objects, unions, and interfaces and bring additional capabilities that are missing in theGQL type system.

## Wrapping Types

### Maybe

Unlike GQL in Iris type system, each type is required.
To model optional types, we use the type `Maybe`, where the type `Maybe<t>` receives the type parameter `t` and returns the type `t | null`. The suffix `?` can be used as an abbreviation for the type `Maybe`.

- required type: `Type`
- optional type: `Type?`

### Lists and Named Lists

As an extension to the regular lists, Iris offers named lists. These named lists apply some specific constraints to their elements. common examples for named lists are `Set` and `NonEmpty`.

```gql
list Set
```

### ADTs (Algebraic Data Types)

The primary entities of the Iris Type System are ADTs (algebraic data types), where an algebraic type is a type with one or more alternatives, where one alternative may have no fields or multiple fields. ADTs in Iris have their roles, namely data (similar to GQL scalar) and resolver (similar to GQL type) roles.

#### Types, variants and typeVariants

In the Iris we distinguish the following terms.

- **Type**: a type is an independent unit containing one or more variants. a type with a singe variant is called **VariantType**.
- **Variant**: a variant is either a reference of VariantType or collection of fields with corresponding tag. this variant will exist inside the type and can't be referenced by another types.

following schema represents the VariantType `God` which is referenced by Type `Deity`, where Deity has another Variant `Titan`. **Note!** `Titan` exists only inside `Deity`. the `__typename` of variant `Titan` is `Deity.Titan`

```gql
# VariantType
<role> God = { name: String }

# Type
<role> Deity
  = God # reference of VariantType
  | Titan # Variant
    {
      name: String
    }
```

#### \_\_typename

every type variant has field \_\_typename.

- Value `A {}` is equal to `{ __typename: "A" }` and can also be parsed (or serialized) as a string `"A"`.
- server will always serialize `resolver` and `data` types as `{ __typename: "A" }`.
- in each selection, field `__typename` will be always automatically selected.
- on input values with multiple variants user must provide field `__typename`.

### Data types

Data types are typed JSON values. Although they have fields, they represent leaf nodes in the graph, which means that their fields cannot be selected and must not have arguments. That is, all fields of the data type are strictly evaluated and sent to the client. Fields of the data type cannot reference resolver types.

Data types do not represent resolver functions, but the compiler will always guarantee the validity of their values for their type definitions. The GraphQL alternative to Data types are GQL scalars with type guarantee.

#### GQL approximation of data types

**data types** are generalization of enums, scalars and input types. Besides, they can also provide input unions and type-safe scalar values to the client.

```gql
# equivalent to GQL enum
data City = Athens {} | Sparta {}

# equivalent to GQL input object
data Deity = { name: String }

# equivalent to  scalar
data Natural = Int
```

### Resolver types

resolver type is always associated with a resolver function. Resolution fields can be selected and can have arguments. moreover, their fields can reference both resolver and data types.

```gql
data Url = {
  path: String;
  domain: String;
}
resolver Post = {
   # resolver field can use data type as argument and result type.
   sanitizeUrl(format: Url): Url
   # resolver type can use resolver types
   posts: [Post]
}
```

for example type `Post` can use type `Url` however, type `Url` can't.

#### ArgumentsDefinition

iris allows default values only for ArgumentDefinitions.

```gql
resolver Type {
  field(max: Int = 10): [String]
}
```

#### Type guards

Resolver type can specify a guard type that requires any variant of the type to provide fields that are compatible with its fields (similar to GraphQL interfaces).

```gql
data U | GuardType = A | B
```

#### GQL approximation of resolver types

**resolver** types are generalization of GraphQL object , unions and interface types.

```gql
## GQL type
resolver A = { a: Int? }

## GQL Union
resolver T
  | A ## GQL interface
  = B { a: Int }
  | C { a: Int? , b: Float }
```

fragments on inline variants.

```gql
fragment X on T.C {
  a
}
```

### Schema

Iris type system selects the resolvers `Query`, `Mutation` and `Subscription` as corresponding schema operations.
