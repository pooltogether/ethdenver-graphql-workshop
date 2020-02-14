# Building a Dapp Using GraphQL

You can literally cut-and-paste most of the code in this tutorial.  A few spots just show the diff.

# Part 1: Build a simple dApp using Tightbeam

View the [complete dapp source on Github](https://github.com/pooltogether/ethdenver-graphql-workshop-app)

## Step 1: Setup

Create the app

```bash
$ yarn create next-app pool-app
$ cd pool-app
```

Install Ethers, Tightbeam, and Apollo Client

```bash
$ yarn add ethers @pooltogether/tightbeam apollo-boost apollo-link-state graphql graphql-tag @apollo/react-hooks
```

## Step 2: Initialize Apollo Client and Tightbeam

Create a new file `lib/createApolloClient.js`

```javascript
// lib/createApolloClient.js

import { Tightbeam } from '@pooltogether/tightbeam'
import { withClientState } from 'apollo-link-state'
import { InMemoryCache } from 'apollo-cache-inmemory'
import { ApolloClient } from 'apollo-client'
import { ApolloLink } from 'apollo-link'

export function createApolloClient() {
  const tb = new Tightbeam()

  // Create a place to store data client-side
  const cache = new InMemoryCache()

  // Ensure that the default state is set
  cache.writeData(tb.defaultCacheData())

  // Now attach the Tightbeam resolvers
  const stateLink = withClientState({
    cache,
    resolvers: tb.resolvers()
  })

  // Hook up the Tightbeam Multicall for speedy call batching
  const link = ApolloLink.from([
    tb.multicallLink(),
    stateLink
  ])

  // Create the Apollo Client
  return new ApolloClient({
    cache,
    link
  })
}

```

Connect the Apollo Client to React in `pages/index.js`

```javascript
// pages/index.js

import React from 'react'
import { ApolloProvider } from '@apollo/react-hooks'
import { createApolloClient } from '../lib/createApolloClient'

let apolloClient = createApolloClient()

const Home = () => (
  <div>
    <ApolloProvider client={apolloClient}>
      Ready for web3!
    </ApolloProvider>
  </div>
)

export default Home

```

## Step 3: Integrate ABIs 

Create a directory `lib/abis` and import the following abis:

**Dai Stablecoin ABI**

Copy the [Dai ABI on Etherscan](https://etherscan.io/address/0x6b175474e89094c44da98b954eedeac495271d0f#code) and paste into `lib/abis/Dai.json`

**PoolTogether PoolDai ABI**

Copy the [PoolTogether Dai Pool ABI on Etherscan](https://etherscan.io/address/0x932773ae4b661029704e731722cf8129e1b32494#code) and paste into `lib/abis/DaiPool.json`

**Note**: This is the address for the proxy *implementation*, not the proxy itself.

Let's integrate our ABIs into Tightbeam.

Create a file `lib/abiMapping.js`:

```javascript
// lib/abiMapping.js

import { AbiMapping } from '@pooltogether/tightbeam'

import Dai from './abis/Dai'
import DaiPool from './abis/DaiPool'

export const abiMapping = new AbiMapping()

abiMapping.addContract('Dai', 1, '0x6B175474E89094C44Da98b954EedeAC495271d0F', Dai)
abiMapping.addContract('DaiPool', 1, '0x29fe7D60DdF151E5b52e5FAB4f1325da6b2bD958', DaiPool)
```

Now pass the abiMapping to the Tightbeam config in `lib/createApolloClient.js`:

```javascript
// lib/createApolloClient.js

// ...
import { abiMapping } from './abiMapping'

export function createApolloClient() {
  const tb = new Tightbeam({
    abiMapping
  })

  // ...
}
```

## Step 4: Display Token Supplies

Create a new component `components/TokenSupplies.jsx`

```javascript
// components/TokenSupplies.jsx

import { useQuery } from '@apollo/react-hooks'
import gql from 'graphql-tag'
import { ethers } from 'ethers'

const supplyQuery = gql`
  query {
    poolCommittedSupply: call(contract: "DaiPool", fn: "committedSupply") @client
    poolOpenSupply: call(contract: "DaiPool", fn: "openSupply") @client
    daiSupply: call(contract: "Dai", fn: "totalSupply") @client
  }
`

export function TokenSupplies() {
  const { loading, error, data } = useQuery(supplyQuery)

  let result = 'Loading...'
  if (error) {
    result = `Error: ${error.message}`
  } else if (data) {
    result = <div>
      <p>Pool Committed Supply: {ethers.utils.formatEther(data.poolCommittedSupply)}</p>
      <p>Pool Open Supply: {ethers.utils.formatEther(data.poolOpenSupply)}</p>
      <p>Dai Supply: {ethers.utils.formatEther(data.daiSupply)}</p>
    </div>
  }

  return result
}
```

Update `pages/index` to include the new component:

```javascript
// pages/index.js

// ... imports
import { TokenSupplies } from '../components/TokenSupplies'

const Home = () => (
  <div>
    <ApolloProvider client={apolloClient}>
      Ready for web3!
      <TokenSupplies />
    </ApolloProvider>
  </div>
)
// ... exports
```

# Part 2: Integrate The Graph Protocol

View the [complete subgraph source on Github](https://github.com/pooltogether/ethdenver-graphql-workshop-subgraph)

## Step 1: Setup

Move to another directory (i.e. not the front-end dir)

Make sure Graph Protocol is installed

```bash
$ yarn global add @graphprotocol/graph-cli
```

Init the project

```bash
$ graph init --from-contract 0x932773ae4b661029704e731722cf8129e1b32494 asselstine/pooltogether-churn pooltogether-churn
```

Fix the address so that it will point to the proxy, and set the starting block to the block before the contract was created.

```yaml
source:
  address: "0x29fe7D60DdF151E5b52e5FAB4f1325da6b2bD958"
  abi: Contract
  startBlock: 9133722
```

The `startBlock` field is very important to make your subgraph speedy.

## Step 6: Define Your Schema

Let's begin by thinking about some questions we'd like answered.

The question of churn:

How many users leave each prize?  How many new users are added for each prize?

Let's define the schema in `schema.graphql`:

```graphql
type PoolPrize @entity {
  id: ID!

  drawId: BigInt!
  
  depositCount: BigInt!
  depositAmount: BigInt!

  withdrawalCount: BigInt!
  withdrawalAmount: BigInt!
}
```

Now re-generate the bindings:

```
$ yarn codegen
```

## Step 2: Process Events

The Graph Protocol can process Ethereum logs, new blocks, or even calls.  We're going to focus on Events.

Let's start by clearing out the boilerplate in `src/mapping.ts`:

```typescript
// remove this line
import { ExampleEntity } from "../generated/schema"

// gut the following function so that it is empty
export function handleAdminAdded(event: AdminAdded): void {}
```

I have the benefit of knowing of how PoolTogether works, but I'll give a quick rundown here:

PoolTogether is a prize linked savings account.  That means that a bunch of people deposit into a Pool that generates interest using the deposits and periodically that interest is awarded as a prize to a randomly selected participant.

Each prize goes through two phases to prevent people from gaming the system (i.e. depositing at the last second):

1. The Prize is opened.  People can now deposit to be eligible for this prize.
2. The Prize is committed.  No new deposits are accepted.  If people withdraw they become ineligible for the prize.
3. The Prize is awarded to an eligible winner!

With that in mind, let's consider our schema.  We defined a PoolPrize entity that counts it's deposits.  There is an event called `Opened` that is emitted when the prize is opened.  Let's start there in `src/mapping.ts`:

```typescript
// src/mapping.ts

// ...
import { PoolPrize } from '../generated/schema'

export function handleOpened(event: Opened): void {
  let prize = new PoolPrize(event.params.drawId.toHexString())
  prize.drawId = event.params.drawId
  prize.depositCount = BigInt.fromI32(0)
  prize.depositAmount = BigInt.fromI32(0)
  prize.withdrawalCount = BigInt.fromI32(0)
  prize.withdrawalAmount = BigInt.fromI32(0)
  prize.save()
}

// ...
```

Now we need to count users deposits into the open Prize:

```typescript
// src/mapping.ts

// ...

export function handleDeposited(event: Deposited): void { 
  let contract = Contract.bind(event.address)
  let drawId = contract.currentOpenDrawId()

  let prize = PoolPrize.load(drawId.toHexString())
  prize.depositCount = prize.depositCount.plus(BigInt.fromI32(1))
  prize.depositAmount = prize.depositAmount.plus(event.params.amount)
  prize.save()
}

// ...
```

Now let's count their withdrawals:

```typescript
// src/mapping.ts

// ...

export function handleWithdrawn(event: Withdrawn): void {
  let contract = Contract.bind(event.address)
  let drawId = contract.currentOpenDrawId()

  let prize = PoolPrize.load(drawId.toHexString())
  prize.withdrawalCount = prize.withdrawalCount.plus(BigInt.fromI32(1))
  prize.withdrawalAmount = prize.withdrawalAmount.plus(event.params.amount)
  prize.save()
}

// ...
```

Great!  We're done.

## Step 3: Deploying

To deploy using The Graph hosted infrastructure, you'll need to go create a subgraph using [the explorer](https://thegraph.com/explorer).

Log in and create one.

Now let's add a convenience script to auth in `package.json`:

```json
{
  "scripts": {
    "auth": "graph auth https://api.thegraph.com/deploy/",
  }
}
```

First we authorize ourselves

```bash
$ yarn auth <your access token>
```

Then we deploy

```bash
$ yarn deploy
```

## Step 4: Integration

Let's jump back to our previous project.

First we'll update our Apollo Client to point to the Graph Protocol endpoint in `lib/createApolloClient.js`:

```javascript
// lib/createApolloClient.js

// ...
import { createHttpLink } from 'apollo-link-http'
// ...

export function createApolloClient() {
  // ...

  const link = ApolloLink.from([
    tb.multicallLink(),
    stateLink,
    createHttpLink({
      uri: 'https://api.thegraph.com/subgraphs/name/asselstine/pooltogether-churn',

      // this is a hack because we're using Next.js
      fetch: (typeof window !== 'undefined') ? window.fetch : () => {}
    })
  ])

  // ...
}
```

And a component to view the data in `components/Prizes.jsx`:

```javascript
// components/Prizes.jsx

import { useQuery } from '@apollo/react-hooks'
import { ethers } from 'ethers'
import gql from 'graphql-tag'

const prizesQuery = gql`
  query {
    poolPrizes {
      id
      depositCount
      depositAmount
      withdrawalCount
      withdrawalAmount
    }
  }
`

export function Prizes() {
  const { loading, error, data } = useQuery(prizesQuery)

  let result = 'Loading...'
  if (error) {
    result = `Error: ${error.message}`
  } else if (data) {
    result = (
      <table>
        <thead>
          <tr>
            <td>
              Prize Id
            </td>
            <td>
              Deposit Count
            </td>
            <td>
              Withdrawal Count
            </td>
            <td>
              Deposit Amount
            </td>
            <td>
              Withdrawal Amount
            </td>
          </tr>
        </thead>
        <tbody>
          {data.poolPrizes.map(poolPrize => (
            <tr key={poolPrize.id.toString()}>
              <td>
                {poolPrize.id.toString()}
              </td>
              <td>
                {poolPrize.depositCount.toString()}
              </td>
              <td>
                {poolPrize.withdrawalCount.toString()}
              </td>
              <td>
                {ethers.utils.formatEther(poolPrize.depositAmount, {commify: true, pad: true})}
              </td>
              <td>
                {ethers.utils.formatEther(poolPrize.withdrawalAmount, {commify: true, pad: true})}
              </td>
            </tr>
          ))}
        </tbody>
      </table>
    )
  }

  return result
}
```

...and add the component to the `pages/index.js`:

```javascript
// ...
import { Prizes } from '../components/Prizes'
// ...

const Home = () => (
  <div>
    <ApolloProvider client={apolloClient}>
      Ready for web3!
      <TokenSupplies />
      <Prizes />
    </ApolloProvider>
  </div>
)
```

And we're done!
