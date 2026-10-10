# RetroRebate

<!-- three.ws:badges -->
[![GitHub stars](https://img.shields.io/github/stars/nirholas/retro-rebate?style=flat&logo=github)](https://github.com/nirholas/retro-rebate/stargazers) [![License](https://img.shields.io/github/license/nirholas/retro-rebate?style=flat)](https://github.com/nirholas/retro-rebate/blob/HEAD/LICENSE) [![Last commit](https://img.shields.io/github/last-commit/nirholas/retro-rebate?style=flat)](https://github.com/nirholas/retro-rebate/commits) [![PRs welcome](https://img.shields.io/badge/PRs-welcome-brightgreen?style=flat)](https://github.com/nirholas/retro-rebate/pulls) [![AI agent friendly](https://img.shields.io/badge/AI%20agents-AGENTS.md%20%2B%20llms.txt-6d5dfc?style=flat)](https://github.com/nirholas/retro-rebate/blob/HEAD/AGENTS.md)
<!-- /three.ws:badges -->


**A volume rebate the pool pays out of its own flow, settled on-chain at the end of each epoch, with no Merkle root, no off-chain calculation and nobody who can decline to publish it.**

A production Uniswap v4 hook. It holds no funds and takes no fee for itself. No owner, no pause switch, no upgrade path.

- **Site:** https://retro-rebate.pages.dev
- **Catalogue:** https://hookforge.pages.dev
- **Contract:** [`src/hooks/RetroRebateHook.sol`](src/hooks/RetroRebateHook.sol)
- **Licence:** Apache-2.0

## How it works

Volume rebates are how every venue rewards its best flow, and on-chain they are almost always a lie about where the computation happens. A team totals volume off-chain, posts a Merkle root, and traders claim against it. The rebate is real; the decentralisation is not.

Whoever runs the script chooses who is in the tree, can publish late, can publish never, and is the only party who can tell you whether the number is right. The reason it is done that way is that per-trader accounting looks expensive, and in a continuous scheme it is: paying a share of a pot that is still growing means either an accumulator per trader or an unbounded loop. Epochs make it cheap.

The hook skims `skimBps` of every swap into that epoch's pot and records the volume each account traded in it, which is two storage writes. Once an epoch has closed, both numbers are final, so a claim is one multiplication and a transfer: payout = pot(epoch) * volume(epoch, account) / totalVolume(epoch) There is nothing to publish, nothing to compute off-chain, and nothing anybody can withhold. A trader who never claims simply leaves their share, and because a closed epoch's pot is fixed, that costs nobody else anything.

Volume is attributed to whatever address the swap names in `hookData`, and this deliberately needs no signature. Naming somebody else credits them with volume you paid for, so the only thing a forged attribution achieves is giving away your own rebate. A swap that names nobody credits the router it came through, which lets a router run the rebate for its users and split it however it likes.

## Prior art

Volume rebates, referral fees and loyalty discounts all exist as hooks, and all of them either discount at the point of trade (which cannot depend on a total that is not known yet) or settle against an off-chain Merkle root. Doing the accounting on-chain in closed epochs, so the rebate is a division anybody can verify and nobody can withhold, is the contribution here.

## Where it does not help

The rebate arrives after the epoch ends, so it is worth less than the same value taken off the trade, and a trader who wants a discount now is better served by a hook that discounts now. Epoch length is a real tradeoff rather than a parameter to shrug at: short epochs pay out sooner and let a single large trade dominate a small pot, long ones smooth that out and make the rebate feel remote.

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
| `AlreadyClaimed(uint256)` | This account has already taken its share of that epoch. |
| `EpochNotClosed(uint256)` | The epoch is still running, so its pot and its total are not final. |
| `HookFeeTooLarge()` | Fee is higher than the maximum allowed fee. |
| `InvalidEpoch()` | An epoch length of zero would make every swap its own epoch and every pot a rounding error. |
| `NothingToClaim(uint256)` | The account traded nothing in that epoch. |
| `PayoutNotPoolManager()` | Only the `PoolManager` may drive the callback. Named distinctly because `BaseHook` declares its own. |
| `SkimTooLarge()` | The skim must leave the swap worth doing. |
| `WrongPool()` | This hook serves one pool, bound at its first initialization. |

## The callbacks it claims

Uniswap v4 reads a hook's permissions from the low fourteen bits of its own address, which is why deploying one means mining a CREATE2 salt. This hook claims 3 of the fourteen:

- `afterInitialize`
- `afterSwap`
- `afterSwapReturnsDelta`

Mask: `0x1044`, so every deployment of this hook has an address ending in those bits.

## It says what it is, on-chain

Every hook in this family implements `IHookMetadata`: four view functions that let an indexer, a wallet, a router or an agent identify a hook from its address alone, with no registry in the loop.

```bash
cast call $HOOK "hookName()(string)"    # RetroRebate
cast call $HOOK "hookVersion()(string)" # 1.0.0
cast call $HOOK "specURI()(string)"     # the machine-readable manifest
cast call $HOOK "hookTags()(string[])"  # rebate, volume, epochs, no-merkle, no-admin
```

The manifest this repository ships as [`hook.json`](hook.json) is what `specURI()` points at.

## Build and test

```bash
git clone --recurse-submodules https://github.com/nirholas/retro-rebate
cd retro-rebate
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

[![Star History Chart](https://api.star-history.com/svg?repos=nirholas/retro-rebate&type=Date)](https://www.star-history.com/#nirholas/retro-rebate&Date)

<!-- three.ws:growth -->
## Support the project

If retro-rebate saves you time, **[star it on GitHub](https://github.com/nirholas/retro-rebate)**. Stars are how other developers and AI agents find the repositories worth trusting, and they cost you one click.

Know someone who would use it? [Post on X](https://twitter.com/intent/tweet?text=retro-rebate%3A%20A%20volume%20rebate%20the%20pool%20pays%20out%20of%20its%20own%20flow%2C%20settled%20on-chain%20at%20the%20end%20of%20each%20epoch%2C%20with&url=https%3A%2F%2Fgithub.com%2Fnirholas%2Fretro-rebate) · [Share on Bluesky](https://bsky.app/intent/compose?text=retro-rebate%3A%20A%20volume%20rebate%20the%20pool%20pays%20out%20of%20its%20own%20flow%2C%20settled%20on-chain%20at%20the%20end%20of%20each%20epoch%2C%20with%20https%3A%2F%2Fgithub.com%2Fnirholas%2Fretro-rebate) · [Share on LinkedIn](https://www.linkedin.com/sharing/share-offsite/?url=https%3A%2F%2Fgithub.com%2Fnirholas%2Fretro-rebate) · [Submit to Hacker News](https://news.ycombinator.com/submitlink?u=https%3A%2F%2Fgithub.com%2Fnirholas%2Fretro-rebate&t=retro-rebate%3A%20A%20volume%20rebate%20the%20pool%20pays%20out%20of%20its%20own%20flow%2C%20settled%20on-chain%20at%20the%20end%20of%20each%20epoch%2C%20with) · [Share on Reddit](https://www.reddit.com/submit?url=https%3A%2F%2Fgithub.com%2Fnirholas%2Fretro-rebate&title=retro-rebate%3A%20A%20volume%20rebate%20the%20pool%20pays%20out%20of%20its%20own%20flow%2C%20settled%20on-chain%20at%20the%20end%20of%20each%20epoch%2C%20with)

## Built for AI agents too

Coding agents and LLM tooling can read this repo directly: [AGENTS.md](./AGENTS.md), [llms.txt](./llms.txt), [llms-full.txt](./llms-full.txt). Point an agent at `https://github.com/nirholas/retro-rebate` and it has the context it needs.

## More from the same author

- [All repositories by nirholas](https://github.com/nirholas/nirholas#readme): the full catalog, grouped by topic
- [three.ws](https://three.ws): the platform for 3D AI agents with Solana wallets, a skill marketplace and x402 payments
- Questions or ideas: [open an issue](https://github.com/nirholas/retro-rebate/issues) or [start a discussion](https://github.com/nirholas/retro-rebate/discussions)

## Contributors

[![Contributors](https://contrib.rocks/image?repo=nirholas/retro-rebate)](https://github.com/nirholas/retro-rebate/graphs/contributors)

<!-- /three.ws:growth -->
