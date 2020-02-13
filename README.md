# ETHDenver - Tutorial 1: Tightbeam

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

  const link = ApolloLink.from([
    tb.multicallLink(),
    stateLink
  ])

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
