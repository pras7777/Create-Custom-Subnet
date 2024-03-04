<p align="center">
  <img width="90%" alt="tokenvm" src="assets/logo.png">
</p>
<p align="center">
  Mint, Transfer, and Trade User-Generated Tokens, All On-Chain
</p>
<p align="center">
  <a href="https://github.com/ava-labs/hypersdk/actions/workflows/tokenvm-static-analysis.yml"><img src="https://github.com/ava-labs/hypersdk/actions/workflows/tokenvm-static-analysis.yml/badge.svg" /></a>
  <a href="https://github.com/ava-labs/hypersdk/actions/workflows/tokenvm-unit-tests.yml"><img src="https://github.com/ava-labs/hypersdk/actions/workflows/tokenvm-unit-tests.yml/badge.svg" /></a>
  <a href="https://github.com/ava-labs/hypersdk/actions/workflows/tokenvm-sync-tests.yml"><img src="https://github.com/ava-labs/hypersdk/actions/workflows/tokenvm-sync-tests.yml/badge.svg" /></a>
  <a href="https://github.com/ava-labs/hypersdk/actions/workflows/tokenvm-load-tests.yml"><img src="https://github.com/ava-labs/hypersdk/actions/workflows/tokenvm-load-tests.yml/badge.svg" /></a>
</p>

---

We created the [`tokenvm`](./examples/tokenvm) to showcase how to use the
`hypersdk` in an application most readers are already familiar with, token minting
and token trading. The `tokenvm` lets anyone create any asset, mint more of
their asset, modify the metadata of their asset (if they reveal some info), and
burn their asset. Additionally, there is an embedded on-chain exchange that
allows anyone to create orders and fill (partial) orders of anyone else. To
make this example easy to play with, the `tokenvm` also bundles a powerful CLI
tool and serves RPC requests for trades out of an in-memory order book it
maintains by syncing blocks.

If you are interested in the intersection of exchanges and blockchains, it is
definitely worth a read (the logic for filling orders is < 100 lines of code!).

## Status
`tokenvm` is considered **ALPHA** software and is not safe to use in
production. The framework is under active development and may change
significantly over the coming months as its modules are optimized and
audited.

## Features
### Arbitrary Token Minting
The basis of the `tokenvm` is the ability to create, mint, and transfer user-generated
tokens with ease. When creating an asset, the owner is given "admin control" of
the asset functions and can later mint more of an asset, update its metadata
(during a reveal for example), or transfer/revoke ownership (if rotating their
key or turning over to their community).

Assets are a native feature of the `tokenvm` and the storage engine is
optimized specifically to support their efficient usage (each balance entry
requires only 72 bytes of state = `assetID|publicKey=>balance(uint64)`). This
storage format makes it possible to parallelize the execution of any transfers
that don't touch the same accounts. This parallelism will take effect as soon
as it is re-added upstream by the `hypersdk` (no action required in the
`tokenvm`).

### Trade Any 2 Tokens
What good are custom assets if you can't do anything with them? To showcase the
raw power of the `hypersdk`, the `tokenvm` also provides support for fully
on-chain trading. Anyone can create an "offer" with a rate/token they are
willing to accept and anyone else can fill that "offer" if they find it
interesting. The `tokenvm` also maintains an in-memory order book to serve over
RPC for clients looking to interact with these orders.

Orders are a native feature of the `tokenvm` and the storage engine is
optimized specifically to support their efficient usage (just like balances
above). Each order requires only 152 bytes of
state = `orderID=>inAsset|inTick|outAsset|outTick|remaining|owner`. This
storage format also makes it possible to parallelize the execution of any fills
that don't touch the same order (there may be hundreds or thousands of orders
for the same pair, so this stil allows parallelization within a single pair
unlike a pool-based trading mechanism like an AMM). This parallelism will take
effect as soon as it is re-added upstream by the `hypersdk` (no action required
in the `tokenvm`).

#### In-Memory Order Book
To make it easier for clients to interact with the `tokenvm`, it comes bundled
with an in-memory order book that will listen for orders submitted on-chain for
any specified list of pairs (or all if you prefer). Behind the scenes, this
uses the `hypersdk's` support for feeding accepted transactions to any
`hypervm` (where the `tokenvm`, in this case, uses the data to keep its
in-memory record of order state up to date). The implementation of this is
a simple max heap per pair where we arrange best on the best "rate" for a given
asset (in/out).

#### Sandwich-Resistant
Because any fill must explicitly specify an order (it is up the client/CLI to
implement a trading agent to perform a trade that may span multiple orders) to
interact with, it is not possible for a bot to jump ahead of a transaction to
negatively impact the price of your execution (all trades with an order occur
at the same price). The worst they can do is to reduce the amount of tokens you
may be able to trade with the order (as they may consume some of the remaining
supply).

Not allowing the chain or block producer to have any control over what orders
a transaction may fill is a core design decision of the `tokenvm` and is a big
part of what makes its trading support so interesting/useful in a world where
producers are willing to manipulate transactions for their gain.

#### Partial Fills and Fill Refunds
Anyone filling an order does not need to fill an entire order. Likewise, if you
attempt to "overfill" an order the `tokenvm` will refund you any extra input
that you did not use. This is CRITICAL to get right in a blockchain-context
because someone may interact with an order just before you attempt to acquire
any remaining tokens...it would not be acceptable for all the assets you
pledged for the fill that weren't used to disappear.

#### Expiring Fills
Because of the format of `hypersdk` transactions, you can scope your fills to
be valid only until a particular time. This enables you to go for orders as you
see fit at the time and not have to worry about your fill sitting around until you
explicitly cancel it/replace it.

### Avalanche Warp Support
We take advantage of the Avalanche Warp Messaging (AWM) support provided by the
`hypersdk` to enable any `tokenvm` to send assets to any other `tokenvm` without
relying on a trusted relayer or bridge (just the validators of the `tokenvm`
sending the message).

By default, a `tokenvm` will accept a message from another `tokenvm` if 80% of
the stake weight of the source has signed it. Because each imported asset is
given a unique `AssetID` (hash of `sourceChainID + sourceAssetID`), it is not
possible for a malicious/rogue Subnet to corrupt token balances imported from
other Subnets with this default import setting. `tokenvms` also track the
amount of assets exported to all other `tokenvms` and ensure that more assets
can't be brought back from a `tokenvm` than were exported to it (prevents
infinite minting).

To limit "contagion" in the case of a `tokenvm` failure, we ONLY allow the
export of natively minted assets to another `tokenvm`. This means you can
transfer an asset between two `tokenvms` A and B but you can't export from
`tokenvm` A to `tokenvm` B to `tokenvm` C. This ensures that the import policy
for an external `tokenvm` is always transparent and is never inherited
implicitly by the transfers between other `tokenvms`. The ability to impose
this restriction (without massively driving up the cost of each transfer) is
possible because AWM does not impose an additional overhead per Subnet
connection (no "per connection" state to maintain). This means it is just as
cheap/scalable to communicate with every other `tokenvm` as it is to only
communicate with one.

Lastly, the `tokenvm` allows users to both tip relayers (whoever sends
a transaction that imports their message) and to swap for another asset when
their message is imported (so they can acquire fee-paying tokens right when
they arrive).

You can see how this works by checking out the [E2E test suite](./tests/e2e/e2e_test.go) that
runs through these flows.

## Demos
Someone: "Seems cool but I need to see it to really get it."
Me: "Look no further."

The first step to running these demos is to launch your own `tokenvm` Subnet. You
can do so by running the following command from this location (may take a few
minutes):
```bash
./scripts/run.sh;
```

_By default, this allocates all funds on the network to
`token1rvzhmceq997zntgvravfagsks6w0ryud3rylh4cdvayry0dl97nsjzf3yp`. The private
key for this address is
`0x323b1d8f4eed5f0da9da93071b034f2dce9d2d22692c172f3cb252a64ddfafd01b057de320297c29ad0c1f589ea216869cf1938d88c9fbd70d6748323dbf2fa7`.
For convenience, this key has is also stored at `demo.pk`._

_If you don't need 2 Subnets for your testing, you can run `MODE="run-single"
./scripts/run.sh`._

To make it easy to interact with the `tokenvm`, we implemented the `token-cli`.
Next, you'll need to build this. You can use the following command from this location
to do so:
```bash
./scripts/build.sh
```

_This command will put the compiled CLI in `./build/token-cli`._

Lastly, you'll need to add the chains you created and the default key to the
`token-cli`. You can use the following commands from this location to do so:
```bash
./build/token-cli key import demo.pk
./build/token-cli chain import-anr
```

_`chain import-anr` connects to the Avalanche Network Runner server running in
the background and pulls the URIs of all nodes tracking each chain you
created._

### Mint and Trade
#### Step 1: Create Your Asset
First up, let's create our own asset. You can do so by running the following
command from this location:
```bash
./build/token-cli action create-asset
```

When you are done, the output should look something like this:
```
database: .token-cli
address: token1rvzhmceq997zntgvravfagsks6w0ryud3rylh4cdvayry0dl97nsjzf3yp
chainID: Em2pZtHr7rDCzii43an2bBi1M2mTFyLN33QP1Xfjy7BcWtaH9
metadata (can be changed later): MarioCoin
continue (y/n): y
✅ txID: 27grFs9vE2YP9kwLM5hQJGLDvqEY9ii71zzdoRHNGC4Appavug
```

_`txID` is the `assetID` of your new asset._

The "loaded address" here is the address of the default private key (`demo.pk`). We
use this key to authenticate all interactions with the `tokenvm`.

#### Step 2: Mint Your Asset
After we've created our own asset, we can now mint some of it. You can do so by
running the following command from this location:
```bash
./build/token-cli action mint-asset
```

When you are done, the output should look something like this (usually easiest
just to mint to yourself).
```
database: .token-cli
address: token1rvzhmceq997zntgvravfagsks6w0ryud3rylh4cdvayry0dl97nsjzf3yp
chainID: Em2pZtHr7rDCzii43an2bBi1M2mTFyLN33QP1Xfjy7BcWtaH9
assetID: 27grFs9vE2YP9kwLM5hQJGLDvqEY9ii71zzdoRHNGC4Appavug
metadata: MarioCoin supply: 0
recipient: token1rvzhmceq997zntgvravfagsks6w0ryud3rylh4cdvayry0dl97nsjzf3yp
amount: 10000
continue (y/n): y
✅ txID: X1E5CVFgFFgniFyWcj5wweGg66TyzjK2bMWWTzFwJcwFYkF72
```

#### Step 3: View Your Balance
Now, let's check that the mint worked right by checking our balance. You can do
so by running the following command from this location:
```bash
./build/token-cli key balance
```

When you are done, the output should look something like this:
```
database: .token-cli
address: token1rvzhmceq997zntgvravfagsks6w0ryud3rylh4cdvayry0dl97nsjzf3yp
chainID: Em2pZtHr7rDCzii43an2bBi1M2mTFyLN33QP1Xfjy7BcWtaH9
assetID (use TKN for native token): 27grFs9vE2YP9kwLM5hQJGLDvqEY9ii71zzdoRHNGC4Appavug
metadata: MarioCoin supply: 10000 warp: false
balance: 10000 27grFs9vE2YP9kwLM5hQJGLDvqEY9ii71zzdoRHNGC4Appavug
```

## Future Work
_If you want to take the lead on any of these items, please
[start a discussion](https://github.com/ava-labs/hypersdk/discussions) or reach
out on the Avalanche Discord._

* Add more config options for determining which order books to store in-memory
* Add option to CLI to fill up to some amount of an asset as long as it is
  under some exchange rate (trading agent command to provide better UX)
* Add expiring order support (can't fill an order after some point in time but
  still need to explicitly close it to get your funds back -> async cleanup is
  not a good idea)
* Add lockup fee for creating a Warp Message and ability to reclaim the lockup
  with a refund action (this will allow for "user-driven" acks on
  messages, which will remain signable and in state until a refund action is
  issued)

<br>
<br>
<br>
<p align="center">
  <a href="https://github.com/ava-labs/hypersdk"><img width="40%" alt="tokenvm" src="assets/hypersdk.png"></a>
</p>
