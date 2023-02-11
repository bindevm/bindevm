# BindEVM (WIP)

Code-first EVM contract entity mapper

## Planned Features
- Contract-first mapper or generated from Typechain
- ERC Interface blocks
- Query interface to The Graph


## Latest version

`0.1.0`

## Installing

### Ethereum
`npm i -S @bindevm/bindevm @bindevm/ethereum-web3`

## What is bindevm

BindEVM is a contract entity mapper, which annotates a Solidity contract based from its generated ABI. Once a contract is annotated with decorators or auto generated, you can enable it to a blockchain by using a plugin vendor.

The pluggable architecture allows different scenarios:

- Server side applications can use a HSM private key based plugin to sign.
- Client side DApps can use specific client wallet like Comet or Metamask.
- Mobile Dapps

## Examples

### Setup:

```typescript
import { Framework } from '@vechain/connex-framework';
import { DriverNodeJS } from '@vechain/connex.driver-nodejs';

import {
  SolidoModule,
} from '@bindevm/bindevm';
import {
 ConnexPlugin,
 ConnexSettings,
} from '@bindevm/vechain-connex';
import {
 ThorifyPlugin,
 ThorifySettings,
} from '@bindevm/vechain-thorify';
import { EnergyTokenContract, EnergyContractImport } from './EnergyContract';
import Web3 from 'web3';
const { thorify } = require('thorify');

// Create Solido Module
export const module = new SolidoModule(
  [
    {
      name: 'ConnexToken',
      import: EnergyContractImport,
      entity: EnergyTokenContract,
      provider: ConnexPlugin,
    },
    {
      name: 'ThorifyToken',
      import: EnergyContractImport,
      entity: EnergyTokenContract,
      enableDynamicStubs: true,
      provider: ThorifyPlugin,
    }
  ],
);

const privateKey = '0x............';
const chainTag = '0x4a';
const defaultAccount = '0x...........';
const thorUrl = 'http://localhost:8669';

const thor = thorify(new Web3(), thorUrl);

const contracts = module.bindContracts();
const token = contracts.getContract<EnergyTokenContract>('ThorifyToken');
token.onReady<ThorifySettings>({
  privateKey,
  thor,
  defaultAccount,
  chainTag
});

(async () => {
  const balance = await token.balanceOf(defaultAccount);
  console.log(balance);
})();
```

### Lazy loading contracts
```typescript
// Create Solido Module
export const module = new SolidoModule(
  [
    {
      name: 'ConnexToken',
      import: EnergyContractImport,
      entity: EnergyTokenContract,
      provider: ConnexPlugin,
    }
  ],
);

module.addContractMapping({
      name: 'ThorifyToken',
      import: EnergyContractImport,
      enableDynamicStubs: true,
      provider: ThorifyPlugin,
});

const privateKey = '0x............';
const chainTag = '0x4a';
const defaultAccount = '0x...........';
const thorUrl = 'http://localhost:8669';

// thorify
const thor = thorify(new Web3(), thorUrl);

// connex framework node driver
const driver = await DriverNodeJS.connect(thorUrl);
const connex = new Framework(driver);
const { wallet } = driver;
wallet.add(PRIVATE_KEY);

const contracts = module.bindContracts({
  'connex': {
    provider: connex,
    options: {
      defaultAccount,
      chainTag,
      // ...connex options
    }
  },
  'thorify': {
    provider: thor,
    {
      privateKey,
      defaultAccount,
      chainTag      
    }
  }
});

// Add new contract and rebind 
module.addContractMapping({
      name: 'AstronautToken',
      import: AstronautToken,
      enableDynamicStubs: true,
      provider: ConnexPlugin,
});

module.rebind();

// Get single contract
const token = contracts.getContract<EnergyTokenContract>('ThorifyToken');
token.connect();

// Get all contracts
interface MyContracts {
  ThorifyToken: EnergyTokenContract,
  ConnexToken: EnergyTokenContract,
}
const { ThorifyToken, ConnexToken }: MyContracts = contracts.connect();

(async () => {
  const balance = await token.balanceOf(defaultAccount);
  console.log(balance);
})();
```



### GetMethod

Returns a function method from a specific provider.

```typescript
class MyContractClass {
  @GetMethod({
    name: 'balanceOf'
  })
  public balanceOfMethod: () => any;
}

// ...
// using the method
const fn = myContractClassInstance.balanceOfMethod();
console.log(fn);
```

### Read

Executes a `call` and returns a response.

```typescript
class MyContractClass {
  @Read()
  public balanceOf: (address: string) => Promise<any>;
}

// ...
// using the read
const balance = await myContractClassInstance.balanceOf(address);
console.log(balance);
```

### Write

Executes a signing flow, which consists of:

* Prepares signing, eg for Connex displays the signing popup window.
* Send

```typescript
class MyContractClass {
  @Write({
    name: 'transfer',
    gas: 1_190_000,
    gasPriceCoef: 0
  })
  public transferMethod: (sendTo: string, wei: BigNumber) => Promise<any>;
}

// ...
// using the write
const tx = await myContractClassInstance.transferMethod(
  '0x........',
  new BigNumber(1 ** 6)
).request({
  gas: 500_000
});
console.log(tx);
```

`request` accepts an object with:

* `gas`: Gas limit
* `gasPriceCoef`: Gas price coefficient
* `from`: From address

### GetEvents

Executes a log event query and returns an array of typed logs.

```typescript
class MyContractClass {
  @GetEvents({
    name: 'Transfer',
    blocks: {
      fromBlock: 0,
      toBlock: 'latest'
    },
    order: 'desc',
    pageOptions: { limit: 10, offset: 0 }
  })
  public getTransferEvents: (
    fnOptions?: EventFilter<any>
  ) => Promise<ThorifyLog[]>;
  // The return type can by ThorifyLog or Connex.Thor.Event depending of the driver used in the contract.
}

// ...
// get the events
// you can pass the same EventFilter object in every call to change the options
const events = await myContractClassInstance.getTransferEvents({
  pageOptions: { limit: 10, offset: 10 }
});
console.log(events);
```

### Short module syntax

When you need to enable any contracts to support more than one provider, use the short module syntax. The provider name will be appended to the name.

```typescript
export const module = new SolidoModule(
  [
    {
      name: 'Token',
      import: EnergyContractImport,
    }
  ],
  ThorifyPlugin, ConnexPlugin, Web3Plugin, TronWebPlugin, QtumPlugin
);

const contracts = module.bindContracts();

const token = contracts.getContract<Web3Plugin>('Web3Token');

const balance = await token.methods.balanceOf();
```

### Dynamic Contract Entities

To let Solido generate Read, Write and Event methods, set `enableDynamicStubs: true` in contract mapping entry and use `GetDynamicContract` to get the contract. The generated stubs are available in `contract.methods`.

```typescript
export const module = new SolidoModule(
  [
    {
      name: 'ThorifyToken',
      import: EnergyContractImport,
      enableDynamicStubs: true
    }
  ],
  ThorifyPlugin
);

const contracts = module.bindContracts();

const token = contracts.getDynamicContract('ThorifyToken');

const balance = await token.methods.balanceOf();

const transferEvent = await token.events.Transfer(); // Returns an event object dependending on provider
```

### Topic Queries

You can query any log event call with a fluent topic query. A contract event signatures are define in `contract.topics`.

```typescript
// build query
let topicQuery = new ConnexSolidoTopic();
topicQuery.topic(0, energy.topics.Transfer.signature);

// set filter options
const filterOptions: EventFilter<any> = {
  pageOptions: {
    limit: 100,
    offset: 0
  },
  topics: topicQuery
};

const logs = await energy.logTransfer(filterOptions);
```

### Connex specific utilities - RxJS operators

#### blockConfirmationUntil

Waits for a block confirmation. Useful for waiting a confirmation and then request the transaction log.

```typescript
const response: any = await energy.logTransfer();
const blockConfirmation = blockConfirmationUntil(response.txid);
const subscription = blockConfirmation
  .pipe(switchMap(_ => response))
  .subscribe((log: any) => {
    // ... code goes here
  });
```

### Plugins

* TODO
