# EpochRebalance

<!-- three.ws:badges -->
[![GitHub stars](https://img.shields.io/github/stars/nirholas/epoch-rebalance?style=flat&logo=github)](https://github.com/nirholas/epoch-rebalance/stargazers) [![License](https://img.shields.io/github/license/nirholas/epoch-rebalance?style=flat)](https://github.com/nirholas/epoch-rebalance/blob/HEAD/LICENSE) [![Last commit](https://img.shields.io/github/last-commit/nirholas/epoch-rebalance?style=flat)](https://github.com/nirholas/epoch-rebalance/commits) [![PRs welcome](https://img.shields.io/badge/PRs-welcome-brightgreen?style=flat)](https://github.com/nirholas/epoch-rebalance/pulls) [![AI agent friendly](https://img.shields.io/badge/AI%20agents-AGENTS.md%20%2B%20llms.txt-6d5dfc?style=flat)](https://github.com/nirholas/epoch-rebalance/blob/HEAD/AGENTS.md)
<!-- /three.ws:badges -->


**An index whose whole future is fixed before anybody deposits: the pool's target allocation glides along a schedule written at deployment, and nobody, including the deployer, can change it afterwards.**

A production Uniswap v4 hook. It holds no funds and takes no fee for itself. No owner, no pause switch, no upgrade path.

- **Site:** https://epoch-rebalance.pages.dev
- **Catalogue:** https://hookforge.pages.dev
- **Contract:** [`src/hooks/EpochRebalanceHook.sol`](src/hooks/EpochRebalanceHook.sol)
- **Licence:** Apache-2.0

## How it works

A weighted pool is already an index fund. Holding `x^w * y^(1-w)` constant keeps the value split at `w` to `1-w` without anybody trading, because arbitrageurs restore the ratio for free every time the price moves. That result is Balancer's and it is why weighted pools are described as self-rebalancing.

What a real index also needs is reconstitution: the target itself changes. Balancer does this with an owner call that starts a gradual weight update, and that is the part worth improving on, for two reasons. The first is trust.

A provider deposits into a 60/40 pool and the owner can make it 20/80 next week. Whatever the governance around that key, the provider's exposure is not theirs to control, and the pool's terms are a promise rather than a property. The second is the rebalance itself.

A weight change is a trade the pool must do, and if it happens as one step the whole trade is available at one price in one block, which is the most arbitrageable shape a rebalance can have. Every index that reconstitutes on a known date pays for this, and it has a name: index front-running. This hook fixes both by fixing the schedule.

Weight checkpoints are set at construction and interpolated linearly between, so the pool's allocation on any future date is computable by anyone from the moment it exists. A provider knows what they are joining for the whole life of the pool. And because the weight moves continuously rather than in a step, the rebalancing flow is spread across the glide instead of concentrated into one block: there is no single moment worth racing to, because at every moment only an instant's worth of the trade is available.

The same mechanism is a liquidity bootstrapping pool when the schedule is two points and the first weight is lopsided, so this covers that case without a separate contract.

## Prior art

Weighted pools as self-rebalancing indices are Balancer's, as are gradual weight updates, which are triggered by an owner. Liquidity bootstrapping pools are the two-point case. Making the entire weight schedule immutable and set before the first deposit, so the pool's future allocation is a property a provider can verify rather than a promise a key holder can revise, is the contribution here, along with the observation that a continuous glide removes the single arbitrageable print a stepped reconstitution creates.

## Where it does not help

The schedule cannot respond to anything. An index that needs to react to a delisting, a merger or a depeg cannot be expressed here, and pretending otherwise by picking a schedule that happens to look right is worse than using a pool with a keeper. The glide also does not eliminate rebalancing cost, it spreads it: the pool still trades into the new weight and still pays for that, it simply does not hand the whole trade to one participant in one block.

## Using it

Uniswap v4 removed `hookData` from `initialize`, so per-pool parameters arrive out of band. Fix them for a pool key whose pool does not exist yet, then initialize. Nobody can change them afterwards, including you.

```solidity
// This hook needs no configuration.

poolManager.initialize(key, startingSqrtPriceX96);
```


### Parameters

This hook takes no per-pool configuration.

## What it reverts with

| Error | Meaning |
| --- | --- |
| `AlreadyInitialized()` | Hook was already initialized. |
| `AmountTooSmall()` | A deposit was too small to mint any shares, or a withdrawal too small to return anything. |
| `ERC20InsufficientAllowance(address,uint256,uint256)` | Indicates a failure with the `spender`’s `allowance`. Used in transfers. |
| `ERC20InsufficientBalance(address,uint256,uint256)` | Indicates an error related to the current `balance` of a `sender`. Used in transfers. |
| `ERC20InvalidApprover(address)` | Indicates a failure with the `approver` of a token to be approved. Used in approvals. |
| `ERC20InvalidReceiver(address)` | Indicates a failure with the token `receiver`. Used in transfers. |
| `ERC20InvalidSender(address)` | Indicates a failure with the token `sender`. Used in transfers. |
| `ERC20InvalidSpender(address)` | Indicates a failure with the `spender` to be approved. Used in approvals. |
| `ExpiredPastDeadline()` | A liquidity modification order was attempted to be executed after the deadline. |
| `InsufficientInitialLiquidity()` | The first deposit must exceed the permanently locked minimum. |
| `InsufficientReserves()` | The pool cannot fill this swap without emptying the side being bought. |
| `InvalidFee()` | The fee must be below 100%. |
| `InvalidNativePayer(address)` | The native currency was settled on behalf of a `payer` other than the contract paying it. |
| `InvalidNativeValue()` | Native currency was not sent with the correct amount. |
| `InvalidSchedule()` | The schedule was empty, too long, out of order, or carried a weight outside the usable band. |
| `LiquidityOnlyViaHook()` | Liquidity was attempted to be added or removed via the `PoolManager` instead of the hook. |
| `NoLiquidity()` | A quote was requested against an empty pool. |
| `PoolNotInitialized()` | Pool was not initialized. |
| `SafeERC20FailedOperation(address)` | An operation with an ERC-20 token failed. |
| `TooMuchSlippage()` | Principal delta of liquidity modification resulted in too much slippage. |

## The callbacks it claims

Uniswap v4 reads a hook's permissions from the low fourteen bits of its own address, which is why deploying one means mining a CREATE2 salt. This hook claims 5 of the fourteen:

- `beforeInitialize`
- `beforeAddLiquidity`
- `beforeRemoveLiquidity`
- `beforeSwap`
- `beforeSwapReturnsDelta`

Mask: `0x2a88`, so every deployment of this hook has an address ending in those bits.

## It says what it is, on-chain

Every hook in this family implements `IHookMetadata`: four view functions that let an indexer, a wallet, a router or an agent identify a hook from its address alone, with no registry in the loop.

```bash
cast call $HOOK "hookName()(string)"    # EpochRebalance
cast call $HOOK "hookVersion()(string)" # 1.0.0
cast call $HOOK "specURI()(string)"     # the machine-readable manifest
cast call $HOOK "hookTags()(string[])"  # curve, index, rebalancing, schedule, no-admin
```

The manifest this repository ships as [`hook.json`](hook.json) is what `specURI()` points at.

## Build and test

```bash
git clone --recurse-submodules https://github.com/nirholas/epoch-rebalance
cd epoch-rebalance
forge build
forge test
```

Foundry 1.7 or newer, Solidity 0.8.26, EVM version `cancun` (Uniswap v4 requires transient storage).

## Deploy

```bash
# Dry run: mines the salt and prints the address without sending anything.
forge script script/Deploy.s.sol --rpc-url $RPC_URL

# For real.
forge script script/Deploy.s.sol --rpc-url $RPC_URL --broadcast --verify
```

Needs `PRIVATE_KEY` in the environment and a funded deployer on the target chain. See [`docs/deploying.md`](docs/deploying.md).

## Status

**Unaudited.** Built to an audited shape, on OpenZeppelin's audited hook bases, and tested against a real `PoolManager`. No third party has reviewed it. Read "where it does not help" above before putting money behind it.

Not affiliated with Uniswap Labs.

## Star History

[![Star History Chart](https://api.star-history.com/svg?repos=nirholas/epoch-rebalance&type=Date)](https://www.star-history.com/#nirholas/epoch-rebalance&Date)

<!-- three.ws:growth -->
## Support the project

If epoch-rebalance saves you time, **[star it on GitHub](https://github.com/nirholas/epoch-rebalance)**. Stars are how other developers and AI agents find the repositories worth trusting, and they cost you one click.

Know someone who would use it? [Post on X](https://twitter.com/intent/tweet?text=epoch-rebalance%3A%20An%20index%20whose%20whole%20future%20is%20fixed%20before%20anybody%20deposits%3A%20the%20pool's%20target%20allocation%20glides&url=https%3A%2F%2Fgithub.com%2Fnirholas%2Fepoch-rebalance) · [Share on Bluesky](https://bsky.app/intent/compose?text=epoch-rebalance%3A%20An%20index%20whose%20whole%20future%20is%20fixed%20before%20anybody%20deposits%3A%20the%20pool's%20target%20allocation%20glides%20https%3A%2F%2Fgithub.com%2Fnirholas%2Fepoch-rebalance) · [Share on LinkedIn](https://www.linkedin.com/sharing/share-offsite/?url=https%3A%2F%2Fgithub.com%2Fnirholas%2Fepoch-rebalance) · [Submit to Hacker News](https://news.ycombinator.com/submitlink?u=https%3A%2F%2Fgithub.com%2Fnirholas%2Fepoch-rebalance&t=epoch-rebalance%3A%20An%20index%20whose%20whole%20future%20is%20fixed%20before%20anybody%20deposits%3A%20the%20pool's%20target%20allocation%20glides) · [Share on Reddit](https://www.reddit.com/submit?url=https%3A%2F%2Fgithub.com%2Fnirholas%2Fepoch-rebalance&title=epoch-rebalance%3A%20An%20index%20whose%20whole%20future%20is%20fixed%20before%20anybody%20deposits%3A%20the%20pool's%20target%20allocation%20glides)

## Built for AI agents too

Coding agents and LLM tooling can read this repo directly: [AGENTS.md](./AGENTS.md), [llms.txt](./llms.txt), [llms-full.txt](./llms-full.txt). Point an agent at `https://github.com/nirholas/epoch-rebalance` and it has the context it needs.

## More from the same author

- [All repositories by nirholas](https://github.com/nirholas/nirholas#readme): the full catalog, grouped by topic
- [three.ws](https://three.ws): the platform for 3D AI agents with Solana wallets, a skill marketplace and x402 payments
- Questions or ideas: [open an issue](https://github.com/nirholas/epoch-rebalance/issues) or [start a discussion](https://github.com/nirholas/epoch-rebalance/discussions)

## Contributors

[![Contributors](https://contrib.rocks/image?repo=nirholas/epoch-rebalance)](https://github.com/nirholas/epoch-rebalance/graphs/contributors)

<!-- /three.ws:growth -->

<!-- nirholas:watermark -->

---

<p align="center">
  Built by <a href="https://github.com/nirholas"><b>nirholas</b></a> · <a href="https://x.com/nichxbt">@nichxbt</a> · <a href="https://github.com/nirholas">github.com/nirholas</a>
</p>

<!-- /nirholas:watermark -->
