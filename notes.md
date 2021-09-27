## The code

### Manifest file

- Is the entry point of the project
- `network`: defines the (para-)chain for connection
- `network.endpoint`: websocket for connecting to the chain
- `network.dictionary`: aggregrator to speed up querying
- `dataSources`: defines a list of data sources, where each source is equipped
  with mappings to the indexed data
- data sources:
  - `kind`: currently only `substrate/Runtime`
  - `startBlock`: the block at which indexing should start
  - `filter`: a network filter, mainly to reuse code between mainnet and testnet
  - `handlers`: a list of handlers
- handlers:
  - `handler`: name of the mapping function
  - `kind`: specifies a handler type
  - `filter` a mapping filter, to avoid triggering on certain
    blocks/events/extrinsics (-> good for performance)

A little yaml trick for reusing definitions:

```yaml
defs:
  mapping: &mymapping
    handlers:
      - handler: handleBlock
        kind: substrate/BlockHandler

dataSources:
  - name: polkadotRuntime
    kind: substrate/Runtime
    filter:
      specName: polkadot
    startBlock: 1000
    mapping: *mymapping
  - name: kusamaRuntime
    kind: substrate/Runtime
    filter:
      specName: kusama
    startBlock: 12000
    mapping: *mymapping
```

### GraphQL schema

- Entities:
  - Required field: `id: ID!`
  - Improve query performance by indexing an entity field with `@index` (not
    allowed on JSON objects): `name: String! @index(unique: true)`

#### Non-primary key field indexing

```graphql
type User @entity {
  id: ID!
  name: String! @index(unique: true)
  title: Title!
}

type Title @entity {
  id: ID!
  name: String! @index(unique: true)
}
```

From code generetion, we will get following convenience methods:

- `User.getByName`
- `User.getByTitleId`
- `Title.getByName`

#### Entity relationships

- One-to-one: Bijectivity reflected in schema, can be unidirectionally linked or
  bidirectionally linked
- One-to-many: Use an array on the the one-side, backlinking is again optional
- Many-to-many: Using a mapping entity which connects the two entities
- Reverse lookups are created by
  `fieldName: [Entity] @derivedFrom(field: derivationFieldName)` (see below)

```graphql
type Account @entity {
  id: ID!
  publicAddress: String!

  # create reverse lookups for sent transfers
  sentTransfers: [Transfer] @derivedFrom(field: "from")
  # create reverse lookups for received transfers
  receivedTransfers: [Transfer] @derivedFrom(field: "to")
}

type Transfer @entity {
  id: ID!
  amount: BigInt
  from: Account!
  to: Account!
}
```

#### JSON types

These types are not put into the DB as entities, but as fields in the table.
They do not require an `id` field.

```graphql
type User @entity {
  id: ID!
  address: Address
}

type Address @jsonField {
  street: String
  zipCode: String
  city: String
}
```

### Mapping functions

A mapping functions defines how to transform chain data into GraphQL entities.

- Mappings are written in the AssemblyScript subset of TypeScript, so they can
  be compiled to wasm
- Mappings are defined in `src/mappings`, exported in `index.ts`, and referenced
  as mapping handlers in the manifest file

## Tooling

- Building: `yarn codegen && yarn build`
  - `codegen` generates typescript from the `schema.graphql`
- Run locally: `docker compose pull && docker compose up`, makes GQL API
  available at `localhost:3000`
- Run

## Gotchas

- [Restrictions to imports](https://doc.subquery.network/create/mapping/#modules-and-libraries)
