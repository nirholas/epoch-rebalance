---
name: use-epoch-rebalance
description: Build, test, integrate and deploy the EpochRebalanceHook Uniswap v4 hook from the epoch-rebalance repo. Use when you need to work with this hook's contract, wire it into a pool, or deploy it with a mined CREATE2 salt.
---

# EpochRebalanceHook

An index whose whole future is fixed before anybody deposits: the pool's target allocation glides along a schedule written at deployment, and nobody, including the deployer, can change it afterwards.

Family: Time. Tags: curve, index, rebalancing, schedule, no-admin. License: Apache-2.0. Source: `src/hooks/EpochRebalanceHook.sol`. The machine-readable manifest is `hook.json` (ABI, errors, permissions, properties).

## Build and test

Foundry project (`foundry.toml`, solc 0.8.26, evm `cancun`). From a clone of the repo:

```bash
git clone https://github.com/nirholas/epoch-rebalance && cd epoch-rebalance
git submodule update --init --recursive   # dependencies live in lib/
forge build
forge test
```

Tests are in `test/EpochRebalanceHook.t.sol`. Use `FOUNDRY_PROFILE=ci forge test` for the larger fuzz run (5000 runs).

## Hook permissions

Uniswap v4 reads a hook's permissions from the low bits of its address. This hook claims: `beforeSwapReturnsDelta`, `beforeSwap`, `beforeRemoveLiquidity`, `beforeAddLiquidity`, `beforeInitialize`. It is not upgradeable and has no custom swap data requirement (`hook.json` `properties`).

## Configuring

This hook has no `configure` call. Its terms are constructor arguments, fixed at deployment:

```solidity
constructor(IPoolManager, Checkpoint[] schedule, uint256 swapFeeBps, string shareName, string shareSymbol)
```

Edit the arguments in `script/Deploy.s.sol` before deploying, since the deployed address depends on them.

## Deploying

A v4 hook must be deployed at an address whose bits match its permissions, so `script/Deploy.s.sol` mines a CREATE2 salt first. The dry run sends nothing:

```bash
export RPC_URL=https://...        # a chain with a Uniswap v4 PoolManager
forge script script/Deploy.s.sol --rpc-url $RPC_URL
```

Broadcasting spends gas from the signer (`PRIVATE_KEY`, or `--account` / `--ledger`) and is irreversible, so show the user the chain, the predicted hook address and the signer, and get an explicit yes before running:

```bash
forge script script/Deploy.s.sol --rpc-url $RPC_URL --broadcast --verify
```

Re-running is safe: a hook already at its mined address is reported and skipped. If the miner fails with `EvmError: OutOfGas`, add `--gas-limit 30000000000`. `--verify` needs `ETHERSCAN_API_KEY`. The chain must be in the address book `src/libraries/Chains.sol`. `script/Deploy.s.sol` fails with "no Uniswap v4 PoolManager known for this chain" otherwise.

## Identifying a deployed hook

```solidity
IHookMetadata(hookAddress).hookName();
IHookMetadata(hookAddress).specURI();   // points at hook.json
```

## Errors you may meet

`AlreadyInitialized()`, `AmountTooSmall()`, `ExpiredPastDeadline()`, `InsufficientInitialLiquidity()`, `InsufficientReserves()`, `InvalidFee()`, `InvalidNativePayer(address)`, `InvalidNativeValue()`, `InvalidPool()`, `InvalidSchedule()`, `LiquidityOnlyViaHook()`, `NoLiquidity()`, `PoolNotInitialized()`, `SafeERC20FailedOperation(address)`, `TooMuchSlippage()`.

## Limitation

The schedule cannot respond to anything. An index that needs to react to a delisting, a merger or a depeg cannot be expressed here, and pretending otherwise by picking a schedule that happens to look right is worse than using a pool with a keeper. The glide also does not eliminate rebalancing cost, it spreads it: the pool still trades into the new weight and still pays for that, it simply does not hand the whole trade to one participant in one block.

More: `docs/integrating.md`, `docs/deploying.md`, and None.
