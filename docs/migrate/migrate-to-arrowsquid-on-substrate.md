---
sidebar_position: 5
title: ArrowSquid for Substrate
description: A step-by-step migration guide for Substrate
---

# Migrate to ArrowSquid (Substrate)

This is a Substrate guide. EVM guide is available [here](/migrate/migrate-to-arrowsquid).

ArrowSquid refers to `@subsquid/substrate-processor` versions `3.x` and above. It is not compatible with the FireSquid archive endpoints. The new `v2` archives are currently being rolled out (see the [Supported networks](/substrate-indexing/supported-networks) page).

The main feature introduced by the ArrowSquid update is the new ability of the [processor](/basics/squid-processor) to ingest unfinalized blocks directly from a network node, instead of waiting for an archive to ingest and serve it first. The processor can now handle forks and rewrite the contents of its database if it happens to have indexed orphaned blocks. This allows Subsquid-based APIs to become near real-time and respond to the on-chain activity with subsecond latency.

[//]: # (!!!! mention any new features here as I discover them)

Other new features include the new streamlined processor configuration interface and automatic retrieval of execution traces. On the implementation side, the way in which data is fetched from archives has been made more efficient.

[//]: # (!!!! add any new examples below)

End-to-end Substrate ArrowSquid examples:
 - [general Substrate indexing](https://github.com/subsquid-labs/squid-substrate-template);
 - [indexing an Ink! contract](https://github.com/subsquid-labs/squid-ink-template/);
 - [working with Frontier EVM pallet data](https://github.com/subsquid-labs/squid-frontier-evm-template).

Here is a step-by-step guide for migrating a squid built with an older SDK version to the post-ArrowSquid tooling.

## Step 1

Update all packages affected by the update:
```bash
npm i @subsquid/substrate-processor@latest @subsquid/typeorm-store@latest
```
```bash
npm i --save-dev @subsquid/substrate-typegen@latest
```
We recommend that you also have `@subsquid/archive-registry` installed. If your squid uses [`file-store`](/store/file-store), please update any related packages to the `@latest` version.

## Step 2

Replace the old archive URL or lookup command with an [ArrowSquid archive lookup for your network](/substrate-indexing/supported-networks) within the `setDataSource()` configuration call. If your squid did not use an RPC endpoint before, find one for your network and supply it to the processor. For Aleph Zero your edit might look like this:
```diff
 processor
   .setDataSource({
-    archive: 'https://aleph-zero.archive.subsquid.io/graphql'
+    archive: lookupArchive('aleph-zero', {release: 'ArrowSquid'}),
+    chain: {
+        url: 'https://aleph-zero-rpc.dwellir.com',
+        rateLimit: 10
+    }
   })
```
We recommend using a private RPC endpoint for the best performance, e.g. from [Dwellir](https://www.dwellir.com). For squids deployed to [Subsquid Cloud](/deploy-squid/quickstart/) you may also consider using our [RPC proxies](/deploy-squid/rpc-proxy).

Your squid will work with just an RPC endpoint, but it will sync significantly slower. With an archive the processor will only use RPC to retrieve metadata **and** sync the few most recent blocks not yet made available by the archive; without it it will retrieve all data from the endpoint.

## Step 3

Next, we have to account for the changes in signatures of the data requesting processor methods

- [`addEvent()`](/substrate-indexing/setup/data-requests/#events),
- [`addCall()`](/substrate-indexing/setup/data-requests/#calls),
- [`addEvmLog()`](/substrate-indexing/specialized/evm/#subscribe-to-evm-events),
- [`addEthereumTransaction()`](/substrate-indexing/specialized/evm/#subscribe-to-evm-transactions),
- [`addContractsContractEmitted()`](/substrate-indexing/specialized/wasm/#processor-options),
- [`addGearMessageEnqueued()`](/substrate-indexing/specialized/gear/#addgearmessageenqueued),
- [`addGearUserMessageSent()`](/substrate-indexing/specialized/gear/#addgearusermessagesent),

Previously, each call of these methods supplied its own fine-grained data fields selector. In the new interface, these calls only request data items, either directly or by relation (for example with the `call` flag for event-requesting methods). Field selection is now done by the new `setFields()` method on a per-item-type basis: once for all `Call`s, once for all `Event`s etc. The setting is processor-wide: for example, all `Call`s returned by the processor will have the same set of available fields, regardless of whether they were requested directly or as related data.

Begin migrating to the new interface by finding all calls to these methods and combining all the field selectors into processor-wide `event`, `call` and `extrinsic` field selectors that request all fields previously requested by individual selectors. Note that `call.args` and `event.args` are now requested by default and can be omitted. When done, add a call to `setFields()` supplying it with the new field selectors.

The new field selector format is fully documented on the [Field selection](/substrate-indexing/setup/field-selection) page.

:::info
Blanket field selections like `{data: {event: {extrinsic: true}}}` are not supported in ArrowSquid. If you used one of these, please find out which exact fields you use in the batch handler and specifically request them.
:::

For example, suppose the processor was initialized with the following three calls:

```typescript
const processor = new SubstrateBatchProcessor()
  .addCall('Balances.transfer_keep_alive', {
    data: {
      call: {
        origin: true,
        args: true
      }
    }
  } as const)
  .addEvent('Balances.Transfer', {
    data: {
      event: {
        args: true,
        extrinsic: {
          hash: true,
          fee: true
        },
        call: {
          success: true,
          error: true
        }
      }
    }
  } as const)
  .addEvmLog(CONTRACT_ADDRESS,
    filter: [[ abi.events.SomeLog.topic ]],
    {
      data: {
        event: {
          extrinsic: {
            hash: true,
            tip: true
          }
        }
      }
    } as const
  )
```
then the new global selectors should be added like this:
```typescript
const processor = new SubstrateBatchProcessor()
  // ... addXXX() calls ...
  .setFields({
    event: {},
    call: {
      origin: true,
      success: true,
      error: true
    },
    extrinsic: {
      hash: true,
      fee: true,
      tip: true
    }
  })
```
Be aware that this operation will not increase the amount of data retrieved from the archive, since previously such coalescence was done under the hood and all fields were retrieved by the processor anyway. In fact, the amount of data should decrease due to a more efficient transfer mechanism employed by ArrowSquid.

There are two old field requests that have no direct equivalent in the new interface:

<details>
<summary>call.parent</summary>

It is currently impossible to request just the parent call. Work around by requesting the full call stack with `stack: true` in the call-requesting configuration calls, then using `.parentCall` property or `getParentCall()` method of `Call` data items to get parent calls.

[//]: # (!!!! Update if the parent call retrieval flag is reintroduced)

</details>

<details>
<summary>event.evmTxHash</summary>

Processor no longer makes EVM transaction hashes explicitly available. **(untested)** Given an `Ethereum.Executed` event `e`, you can get it as `e.args[2] || e.args.transactionHash`.

[//]: # (!!!! Test the workaround)

</details>

## Step 4

Replace the old calls to the data requesting processor methods with calls using the new signatures.

:::warning
The meaning of passing `[]` as a set of parameter values has been changed in the ArrowSquid release: now it _selects no data_. Pass `undefined` for a wildcard selection:
```typescript
.addEvent({name: []}) // selects no events
.addEvent({}) // selects all events
```
:::

Old data request calls will be erased during the process. Make sure to request the appropriate related data with the boolean flags (`call` for event-requesting methods, `events` for call-requesting methods and `extrinsic`, `stack` for both).

Interfaces of data request methods are documented on their respective pages:
- [`addEvent()`](/substrate-indexing/setup/data-requests/#events),
- [`addCall()`](/substrate-indexing/setup/data-requests/#calls),
- [`addEvmLog()`](/substrate-indexing/specialized/evm/#subscribe-to-evm-events),
- [`addEthereumTransaction()`](/substrate-indexing/specialized/evm/#subscribe-to-evm-transactions),
- [`addContractsContractEmitted()`](/substrate-indexing/specialized/wasm/#processor-options),
- [`addGearMessageEnqueued()`](/substrate-indexing/specialized/gear/#addgearmessageenqueued),
- [`addGearUserMessageSent()`](/substrate-indexing/specialized/gear/#addgearusermessagesent).

Here is a fully updated initialization code for the example processor from step 3:
```typescript
const processor = new SubstrateBatchProcessor()
  .addCall({
    name: [ 'Balances.transfer_keep_alive' ]
  })
  .addEvent({
    name: [ 'Balances.Transfer' ],
    call: true,
    extrinsic: true
  })
  .addEvmLog({
    address: [ CONTRACT_ADDRESS ],
    topic0: [ abi.events.SomeLog.topic ],
    extrinsic: true
  })
  .setFields({
    event: {},
    call: {
      origin: true,
      success: true,
      error: true
    },
    extrinsic: {
      hash: true,
      fee: true,
      tip: true
    }
  })

```

## Step 5

Finally, update the batch handler to use the new [batch context](/basics/squid-processor/#batch-context). The main change here is that now `block.items` is split into three separate iterables: `block.calls`, `block.events` and `block.extrinsics`. There are two ways to migrate:

1. If you're in a hurry, use the `orderItems(block: Block)` function from [this snippet](https://gist.github.com/belopash/5d61dcce7739f60d55c4faaec0148282):
   ```typescript title=src/main.ts
   // ...
   // paste the gist here

   processor.run(db, async ctx => {
     // ...
     for (let block of ctx.blocks) {
       for (let item of orderItems(block)) {
         // item processing code should work unchanged
       }
     }
     // ...
   })
   ```

2. Alternatively, rewrite your batch handler using the [new batch context interface](/basics/squid-processor/#batch-context).

See [Block data for Substrate](/substrate-indexing/context-interfaces) for the documentation on Substrate-specific part of batch context.

## Step 6

Rewrite your `typegen.json` in the new style. Here is an example:
```json
{
  "outDir": "src/types",
  "specVersions": "https://v2.archive.subsquid.io/metadata/kusama",
  "pallets": {
    "Balances": {
      "events": [
        "Transfer"
      ],
      "calls": [],
      "storage": [],
      "constants": []
    }
  }
}
```
Note the changes:
1. Archive URL as `"specVersions"` is replaced with an URL of our new metadata service (`"https://v2.archive.subsquid.io/metadata/kusama"`)
2. Requests for data wrappers are now made on a per-pallet basis.

Check out the updated [Substrate typegen documentation page](/substrate-indexing/squid-substrate-typegen). If you used any storage calls, consult [this documentation page](/substrate-indexing/storage-state-calls) for guidance.

Once you're done migrating `typegen.json`, regenerate the wrapper classes with
```bash
sqd typegen
```
or equivalently
```bash
npx squid-substrate-typegen typegen.json
```

## Step 7

Iteratively reconcile any type errors arising when building your squid (e.g. with `sqd build`). If you need to specify the field selection generic type argument explicitly, get it as a `typeof` of the `setFields` argument value:

```ts
import {Block} from '@subsquid/substrate-processor'

const fieldSelection = {
  event: {},
  call: {
    origin: true,
    success: true,
    error: true
  },
  extrinsic: {
    hash: true,
    fee: true,
    tip: true
  }
} as const

type MyBlock = Block<typeof fieldSelection>
// ...
```

At this point your squid should be able to work with the ArrowSquid tooling. If it doesn't, read on.

## Troubleshooting

If these instructions did not work for you, please let us know at the [SquidDevs Telegram chat](https://t.me/HydraDevs).
