# 🎮 NEON ARCADE — Higher / Lower on LitVM

An on-chain retro arcade built for the **LitVM Builders Program**, deployed to **LiteForge Testnet (chain ID 4441)**.

Play a free-to-play Higher/Lower card game, farm XP, claim `HI` governance tokens, build daily GM streaks, earn NFT badges, and govern the protocol through a timelocked on-chain DAO — all in one DApp, with no backend and no off-chain oracle.

---

## Table of Contents

1. [What this is](#what-this-is)
2. [Live deployment](#live-deployment)
3. [Quick start](#quick-start)
4. [Architecture](#architecture)
5. [How the contracts interlock](#how-the-contracts-interlock)
6. [XP & leveling math](#xp--leveling-math)
7. [Governance model](#governance-model)
8. [Repository layout](#repository-layout)
9. [Running the DApp](#running-the-dapp)
10. [Compiling](#compiling)
11. [Deploying from scratch](#deploying-from-scratch)
12. [Verifying on the explorer](#verifying-on-the-explorer)
13. [Tests](#tests)
14. [Scripts reference](#scripts-reference)
15. [Network & tooling](#network--tooling)
16. [Design notes, trust model & known limits](#design-notes-trust-model--known-limits)
17. [Roadmap](#roadmap)
18. [License](#license)

---

## What this is

Neon Arcade is a complete, self-contained demonstration of a consumer DApp on LitVM. Every action a user takes — starting a card game, making a guess, saying `gm`, claiming XP, claiming tokens, voting — is an on-chain transaction. There is:

- **No backend server.** The frontend talks directly to the RPC and to an on-chain leaderboard.
- **No off-chain XP oracle.** `XPTracker` derives XP by reading other contracts' live state with `view` calls.
- **No admin mint.** `HIToken` supply is fixed at deploy time and cannot inflate.

### The three user-facing surfaces

| Surface | What the user does | Contract |
|---|---|---|
| **Arcade tab** | Start a Higher/Lower game, guess higher/lower, chase the top-10 leaderboard | `HigherLowerArcade` |
| **GM Streak tab** | Check in once per UTC day, build a streak, mint badge/milestone NFTs | `GMStreak` → `StreakBadgeNFT`, `GMMilestoneNFT` |
| **DAO page** | Claim `HI` for XP, delegate votes, create/vote/queue/execute proposals | `HIToken`, `ArcadeGovernor`, `ArcadeTimelock` |

A fourth page (`NftPage.jsx`) renders the level-badge gallery with static metadata in `frontend/public/nft/`.

---

## Live deployment

All nine contracts are deployed and verified on LiteForge Testnet.

| Contract | Role | Address |
|---|---|---|
| `HigherLowerArcade` | Card game + leaderboard | `0xb7C31Ae4de6EcA06B1b9df6810b3dC31617f1540` |
| `GMStreak` | Daily check-in streaks | `0xE983F96B36afbE34213aAAC78dF798a8e6dF89a5` |
| `StreakBadgeNFT` | 3-day streak badge (ERC-721) | `0xb4459d67bf34a28B1CBEaCd7806a851fD1B86ff4` |
| `GMMilestoneNFT` | 2/3/5/8/12-day tiers (ERC-721) | `0x82e051d3c2939e64981bF830AF10d15Bf576b52C` |
| `XPTracker` | XP accrual, levels, claims | `0xE7DF5F92f3b257d3968D54ca78c4C973324f9859` |
| `ArcadeLevelNFT` | Visual level badges (ERC-721) | `0xb4Ed9002cf29c80Fe6B615e86ED79F22d14468F1` |
| `HIToken` | `HI` governance token (ERC-20 + votes) | `0x94C1d4CBCf07a1055Aa122B4481A587Fb2E52831` |
| `ArcadeTimelock` | Delayed execution of DAO actions | `0x923ed528a023B0245D6910Dbb1FbDDab3896CcBA` |
| `ArcadeGovernor` | Proposal / vote lifecycle | `0x4f59eBAF4705A45954759834E424eF008D41018c` |

> These same addresses are mirrored in `scripts/verify_all_live.ts`, `scripts/dao_step4_5.ts`, and the frontend's `CONFIG.CONTRACTS`.

---

## Architecture

```
                                    ┌──────────────────┐
                                    │  HigherLowerArcade│  game state:
                                    │  score/best/games/│  players(), topPlayers[]
                                    │  moves            │
                                    └────────┬─────────┘
                                             │ players(address)
                                             │ (read-only)
┌──────────────────┐   badge()/milestone()  ▼
│    GMStreak      │──────────────┐  ┌──────────────┐
│  streakers()     │              │  │  XPTracker   │  computeEarnedXP(user)
│  gm() daily      │──────────────┼─▶│  claimXP()   │  pendingXP / getXPBreakdown
│  badge/milestone │              │  └──────┬───────┘
└───┬──────────┬───┘              │         │ getXPBreakdown()      xp(address)
    │ mints    │ mints            │         ▼                            │
    ▼          ▼                  │  ┌──────────────┐        ┌───────────▼──────────┐
┌────────────┐ ┌──────────────┐   │  │ArcadeLevelNFT│        │      HIToken         │
│StreakBadge │ │GMMilestoneNFT│   │  │ mint(tier)   │        │ ERC-20 + checkpoints │
│   NFT      │ │              │   │  └──────────────┘        │ claimXP() 60k pool   │
└────────────┘ └──────────────┘   │                          └───────────┬──────────┘
                                  │                                      │ token()
                                  │                          ┌───────────▼──────────┐
                                  │                          │   ArcadeGovernor     │
                                  │                          │ propose/vote/queue   │
                                  │                          │      /execute        │
                                  │                          └───────────┬──────────┘
                                  │                                      │ timelock()
                                  │                          ┌───────────▼──────────┐
                                  └─────────────────────────▶│   ArcadeTimelock     │
                                     (treasury, holds 40k HI) │  queue()/execute()   │
                                                             └──────────────────────┘
```

**Dependency order at deploy time** (each arrow = constructor argument):

```
HigherLowerArcade
GMStreak ──────────────► StreakBadgeNFT, GMMilestoneNFT (deployed inside its constructor)
XPTracker        ← HigherLowerArcade, GMStreak
ArcadeLevelNFT   ← XPTracker
ArcadeTimelock   ← deployer (temporary), delay
HIToken          ← XPTracker, treasury (= ArcadeTimelock)
ArcadeGovernor   ← HIToken, ArcadeTimelock
ArcadeTimelock.setGovernor(ArcadeGovernor)   ← final handover
```

---

## How the contracts interlock

### 1. Gameplay → XP (pull-based, no oracle)

`XPTracker` never receives a callback. It **reads** the other contracts on demand:

```solidity
function computeEarnedXP(address user) public view returns (uint256) {
    return games * XP_PER_GAME
         + moves * XP_PER_MOVE
         + totalGM * XP_PER_GM
         + best * XP_PER_BEST
         + badgeCount * XP_PER_BADGE
         + milestoneCount * XP_PER_MILESTONE;
}
```

Consequences worth understanding:

- **State stays consistent by construction.** Play more → earned XP rises. There is no "sync" step that can drift.
- **Claiming is the only write.** `claimXP()` stores `claimedXP = earned` and credits the delta. If nothing new happened, it reverts — so XP cannot be re-claimed by spamming the function.
- **Each claim is a transaction**, which is itself arcade "activity", reinforcing the loop.
- **NFT balances** are read via a `try IERC721Lite(...).balanceOf(user)` with a `catch {}` fallback, so a broken NFT address cannot brick XP. If the badge NFT read returns 0 but `gmStreak.streakers(user).badgeMinted` is true, it falls back to 1 badge worth of XP.

### 2. GM streak → NFTs

`GMStreak.gm()` runs on a UTC-day index (`block.timestamp / 1 days`):

- Same day again → `require(s.lastGM != d, "already gm today")`.
- `lastGM == d - 1` → streak increments; otherwise streak resets to 1.
- At streak ≥ 3, mints **one** `StreakBadgeNFT` per wallet (`badgeMinted` guard).
- On the *exact* tier day (2 / 3 / 5 / 8 / 12), mints a `GMMilestoneNFT` — repeatable tiers, unlimited supply.

`StreakBadgeNFT` and `GMMilestoneNFT` are deployed **inside** the `GMStreak` constructor, so the deployer only passes metadata URIs, and `minter` is automatically `GMStreak`.

### 3. XP → tokens → votes

`HIToken.claimXP()` reads `xpTracker.xp(user).claimedXP` (defensively, via `try/catch`), converts unspent XP at **1 HI per 1 XP**, and clamps the payout twice:

- per-wallet ceiling `MAX_CLAIM_PER_WALLET = 2,000 HI`
- global pool `CLAIM_POOL = 60,000 HI`

It tracks `xpUsed[user]` separately from `claimedFromXP[user]`, so a wallet that hits its 2,000 HI ceiling does not silently lose XP accounting.

Transfers and mints write **per-block checkpoints**, exposing `getPastVotes` / `getPastTotalSupply`. Delegation defaults to self (`delegates[0] == address(0)` → self-delegated).

### 4. Votes → proposals → timelock execution

See [Governance model](#governance-model).

---

## XP & leveling math

Constants in `XPTracker`:

| Constant | Value | Meaning |
|---|---|---|
| `XP_PER_GAME` | 2 | per `startGame()` |
| `XP_PER_MOVE` | 1 | per **correct** guess |
| `XP_PER_GM` | 3 | per daily `gm()` |
| `XP_PER_BEST` | 1 | per all-time best score point |
| `XP_PER_BADGE` | 5 | per `StreakBadgeNFT` held (0/1) |
| `XP_PER_MILESTONE` | 10 | per `GMMilestoneNFT` held |
| `XP_PER_LEVEL` | 25 | level = `1 + earnedXP / 25` |

Worked example — a player who starts 10 games, makes 40 correct guesses on the way to a best of 12, says `gm` 5 times and holds both a badge and 2 milestone NFTs:

```
games      10 ×  2 =  20
moves      40 ×  1 =  40
best       12 ×  1 =  12
gms         5 ×  3 =  15
badge       1 ×  5 =   5
milestones  2 × 10 =  20
                     ────
earned XP          = 112   →  level = 1 + 112/25 = 5
claimable HI       = 112   (well under the 2,000 per-wallet cap)
```

`getXPBreakdown(user)` returns all six components plus `totalEarned`, `claimedXP`, `pending`, `level` and `claimCount` in a single call — the frontend uses this instead of issuing six reads.

---

## Governance model

```
propose ──▶ Pending ──(votingDelay)──▶ Active ──(votingPeriod)──▶ Succeeded ──▶ Queued ──(timelock delay)──▶ Executed
                                        │
                                        └──▶ Defeated (quorum not met, or against ≥ for)

Any state ──(proposer calls cancel)──▶ Cancelled
```

Defaults set in `ArcadeGovernor`:

| Parameter | Value | Notes |
|---|---|---|
| `votingDelay` | 1 block | snapshot block = `startBlock - 1` |
| `votingPeriod` | 7,200 blocks | ≈ 1 day at 12s blocks |
| `proposalThreshold` | `1e18` (1 HI) | read from **past** votes |
| `quorumBps` | 400 (4%) | of `getPastTotalSupply(startBlock - 1)` |
| `ArcadeTimelock.delay` | 86,400 s | 1 day |
| `MAX_OPERATIONS` | 10 | declared bound for future multi-call support |

**Deliberate design choices**

- **Snapshot voting.** `castVote` reads `getPastVotes(voter, startBlock - 1)`, so tokens bought after a proposal opens cannot vote on it and flash-loaned tokens are worthless. `weight == 0` reverts with `NoVotingPower()`.
- **The governor is not the treasury.** The timelock holds the 40,000 HI treasury allocation and enforces the delay; the governor can only *schedule* into it. Parameter changes (`setVotingParams`) are callable **only** by the timelock, i.e. only through a successful proposal.
- **Cancellation is proposer-only** and clears the timelock queue entry if one exists.

**Governance is bootstrapped without free money.** 40,000 of the 100,000 HI supply is placed in the timelock, the other 60,000 is claimable strictly against claimed XP. The deployer holds no privileged supply, so the only way to acquire votes is to actually play.

---

## Repository layout

```
contracts/
  HigherLowerArcade.sol    Higher/Lower game: players(), top-10 leaderboard
  GMStreak.sol             Daily gm check-in; deploys both NFTs in its constructor
  StreakBadgeNFT.sol       Minimal ERC-721, 3-day streak badge
  GMMilestoneNFT.sol       Minimal ERC-721, 2/3/5/8/12 tiers, unlimited supply
  XPTracker.sol            Pure view + claim XP accounting
  ArcadeLevelNFT.sol       Tiered level badge gallery, reads XP level
  HIToken.sol              Fixed-supply ERC-20 with per-block vote checkpoints
  ArcadeTimelock.sol       Delay-enforcing execution target / treasury
  ArcadeGovernor.sol       propose / castVote / queue / execute / cancel
  CodeChecker.sol          Scratch helper used during development
  _verify/                 Snapshots kept for explorer verification
  1_Storage.sol            Remix template scaffolding (unused by the DApp)
  2_Owner.sol               "
  3_Ballot.sol              "
artifacts/                  Compiled ABI + bytecode for every contract
frontend/
  index.html               Static entry point (inline Babel, no build step)
  src/App.jsx              Arcade + GM streak tabs, wallet/chain plumbing
  src/DaoPage.jsx          Governance UI
  src/NftPage.jsx          Level-badge gallery
  src/index.css            Neon arcade theme
  src/nft-page.css         NFT page styles
  public/nft/*.json|*.svg  Metadata + art for level badges 2/3/5/8/12
scripts/
  deploy_with_ethers.ts    Full 9-contract deployment (see below)
  ethers-lib.ts            Small Remix deploy helper
  verify_all_live.ts       Read-only health check of all live contracts
  verify_all_live_fetch.ts Sibling check using raw fetch
  verify_arcade_nft.ts     ArcadeLevelNFT-specific inspection
  check_artifact_provenance.ts  Compares disk artifacts against deployed bytecode
  liteforge_readonly_check.ts   Generic read-only probe
  dao_step4_5.ts           Governor handover + read-back (historic step 4/5)
  xp_diag_4441_readonly.ts XP diagnostics for a specific wallet
  hl_read.ts               Higher/Lower state read helper
  diag.ts                  Live probe of all 9 contracts (code + one getter each)
  diag1.ts                 Governance wiring assertions (token/governor/timelock)
tests/
  ArcadeCore.test.js       Hardhat tests: XP, GM streak, full governance lifecycle
  storage.test.js          Remix template test for Storage
foundry.toml               Foundry config + LiteForge RPC/explorer endpoints
hardhat.config.js          Hardhat: solc 0.8.19, optimiser off, litvm_testnet network
package.json               Dev deps + npm scripts (compile, test, deploy, verify)
.env.example               Every env var the scripts read (copy to .env)
dapp.config.json           QuickDapp binding (ABIs + live addresses)
remix.config.json          Remix workspace configuration
LICENSE                    MIT
```

---

## Quick start

```bash
git clone <repo> && cd neon-arcade
npm install                      # hardhat, ethers, chai, ts-node
cp .env.example .env             # fill PRIVATE_KEY only if you intend to deploy
npm run compile                  # solc 0.8.19, optimiser off
npm test                         # core suite: XP, GM streaks, governance
npm run verify:live              # read-only health check against chain 4441
```

Nothing above needs a key except a fresh deployment. The DApp itself needs no build step — see below.

---

## Running the DApp

The frontend is **pre-built** — no bundler, no `npm install`. `index.html` loads React and Babel from CDN and transforms the JSX in the browser.

```bash
# simplest possible static server
cd frontend
python3 -m http.server 8080
# → http://localhost:8080
```

1. Click **Connect Wallet** (MetaMask).
2. If you are on the wrong network, accept the **Switch to LiteForge** prompt — the app calls `wallet_addEthereumChain` with chain ID `4441` and then `wallet_switchEthereumChain`.
3. Fund the wallet from the [Caldera faucet](https://liteforge.explorer.caldera.xyz).
4. Play. Reads work without a wallet (leaderboard, global stats, NFT lookups); writes require one.

**Contract addresses** are resolved from `CONFIG.CONTRACTS` in `App.jsx`. If you redeploy, update that object and the matching lists in the verify scripts.

---

## Compiling

Contracts are `pragma solidity ^0.8.19` and build cleanly on **0.8.34** (the version configured in this Remix workspace) with optimisation **off**.

- **Remix:** open any contract, `Ctrl+S` (or enable auto-compile). `remix.config.json` pins the workspace settings.
- **Foundry:** `forge build` (see `foundry.toml`; `solc = "0.8.19"`).
- **Hardhat (for the test suite):** needs a `hardhat.config.js` with `solidity: "0.8.19"` or `"0.8.34"` and `paths.sources = "./contracts"`.

Artifacts land in `/artifacts/<Name>.json`; the deployment script loads them from there.

---

## Deploying from scratch

`scripts/deploy_with_ethers.ts` was rewritten from the Remix `Storage` boilerplate into a full deployment script. It:

- loads artifacts from Remix's `fileManager` **or** the local filesystem, so it runs in both environments,
- normalises ethers v5 and v6 (`deployed()` / `waitForDeployment()`),
- deploys all nine contracts in dependency order,
- performs the `ArcadeTimelock.setGovernor(governor)` handover,
- runs twelve post-deploy sanity assertions (wiring + supply + `ROUND_SIZE`),
- prints an address book ready to paste into `CONFIG.CONTRACTS`.

**Configuration** — every value is overridable by environment variable:

| Env var | Default | Used by |
|---|---|---|
| `ARCADE_BADGE_URI` | `https://arcade.example/nft/badge/` | `GMStreak` |
| `ARCADE_MILESTONE_URI` | `https://arcade.example/nft/milestone/` | `GMStreak` |
| `ARCADE_LEVEL_BASE_URI` | `https://arcade.example/nft/` | `ArcadeLevelNFT` |
| `ARCADE_TIMELOCK_DELAY` | `86400` | `ArcadeTimelock` |
| `ARCADE_TREASURY` | *(empty)* → resolved to `ArcadeTimelock` | `HIToken` |

**Run it**

```bash
# Remix: open the file, right-click → Run (Ctrl+Shift+S)
# Node:
npx ts-node scripts/deploy_with_ethers.ts
```

Suggested production URIs point at the static metadata already in `frontend/public/nft/`:

```bash
export ARCADE_LEVEL_BASE_URI="https://<your-host>/nft/"
```

---

## Verifying on the explorer

Verification is standard LiteForge/Caldera Blockscout-style verification — constructor arguments matter, so keep the exact deploy parameters.

1. Open `https://liteforge.explorer.caldera.xyz`.
2. Search the contract address → **Verify & Publish**.
3. Supply the compiler version (`0.8.19` or `0.8.34`), optimisation **off**, and EVM version as compiled.
4. Paste the flattened source or upload the file. For `GMStreak`, remember `StreakBadgeNFT` and `GMMilestoneNFT` are deployed *from within it*, so their verification needs the constructor args `("GM Streak Badge", "GMSB", <badgeURI>)` and `("GM Milestone", "GMMS", <milestoneURI>)`.
5. Constructor arguments:
   - `XPTracker(arcade, gmStreak)`
   - `ArcadeLevelNFT(xpTracker, baseURI)`
   - `ArcadeTimelock(initialGovernor, delay)`
   - `HIToken(xpTracker, treasury)`
   - `ArcadeGovernor(hiToken, timelock)`

`scripts/verify_all_live.ts` confirms live wiring afterwards: chain ID, block number, bytecode length, and key getters for all nine contracts. `scripts/check_artifact_provenance.ts` additionally diffs the on-disk artifact bytecode against what is actually deployed — useful for proving the published source matches the running contract.

---

## Tests

`tests/ArcadeCore.test.js` is the substantive suite (Hardhat + Mocha/Chai, ethers v5 + `hardhat-chai-matchers`):

```bash
npm install
npx hardhat test tests/ArcadeCore.test.js   # or: npm test
```

### `XPTracker` coverage

- immutable `arcade` / `gmStreak` wiring, all seven XP weight constants
- zero XP for a fresh wallet; `claimXP` reverts with `"No new XP - interact with the arcade or GM streak first"`
- 2 XP per game started; combined games/moves/best/GM accrual verified against `players()` + `streakers()`
- `claimXP` emits `XPClaimed`, advances `level`, `totalXP`, `totalClaims`, and cannot be claimed twice for the same activity
- delta-only crediting across repeated claim cycles
- level formula `1 + earned/25` (13 games → 26 XP → level 2)
- per-wallet isolation between Alice and Bob
- `getXPBreakdown` component summation invariant

### `GMStreak` coverage (inside the XP suite)

- 3 XP per `gm()`; same-day second `gm()` reverts `"already gm today"`
- 3 consecutive days → streak 3, badge minted, 34 XP total (9 GM + 5 badge + 2×10 milestone)
- missing a day resets the streak to 1 while `longest` stays honest

### Governance coverage

- token / timelock / governor wiring, timelock governed by the governor
- treasury holds exactly 40,000 HI with matching voting power; total supply 100,000
- below-threshold proposal reverts `BelowThreshold()`
- **full lifecycle**: propose → `Pending` → `Active` → vote (weight asserted) → `Succeeded` → `Queued` → timelock delay → `Executed`, with `arcade.totalEntries()` proving the call actually landed
- double-vote reverts `AlreadyVoted()`; zero-power vote reverts `NoVotingPower()`
- proposer-only cancellation (`NotProposer()`), cancelled proposals cannot queue
- quorum failure → `Defeated`
- `setVotingParams` is timelock-only; `quorum()` = 4% of supply

`tests/storage.test.js` is the untouched Remix template test.

---

## Scripts reference

| Script | Purpose | Needs a signer? |
|---|---|---|
| `deploy_with_ethers.ts` | Deploy all 9 contracts + handover + sanity checks | yes |
| `verify_all_live.ts` | Health-check live chain + all contracts | no |
| `verify_all_live_fetch.ts` | Same, using raw `fetch` against the RPC | no |
| `verify_arcade_nft.ts` | Inspect `ArcadeLevelNFT` levels / URI / supply | no |
| `check_artifact_provenance.ts` | Diff disk bytecode vs deployed bytecode | no |
| `dao_step4_5.ts` | Historic governor handover + governance read-back | yes |
| `xp_diag_4441_readonly.ts` | Per-wallet XP diagnostics on chain 4441 | no |
| `hl_read.ts` | Higher/Lower state snapshot | no |
| `liteforge_readonly_check.ts` | Generic read-only probe | no |
| `diag.ts` | Probes all 9 live contracts: bytecode + one representative getter | no |
| `diag1.ts` | Asserts token → governor → timelock wiring and governor params | no |

Read-only scripts are intentionally wallet-free so reviewers can run them against the live deployment without keys.

---

## Network & tooling

| Parameter | Value |
|---|---|
| Network | LiteForge Testnet (LitVM) |
| Chain ID | `4441` |
| RPC | `https://liteforge.rpc.caldera.xyz/http` |
| Explorer | `https://liteforge.explorer.caldera.xyz` |
| Currency | testnet `ETH` |
| Rollup stack | Caldera |

**Stack**

- Solidity `^0.8.19`, compiled with Remix `0.8.34`, optimisation off
- Ethers.js v6 in the browser frontend (CDN); ethers v5 + Hardhat for the test suite and scripts
- React via CDN + in-browser Babel — zero build step
- Hardhat + Mocha/Chai for tests
- Foundry config present for `forge build` / `cast` workflows

**LitVM specifics**

- Chain ID `4441` is hard-coded in the frontend chain-switch flow and in the verify scripts; `wallet_addEthereumChain` is called before `wallet_switchEthereumChain` so fresh wallets work first try.
- All reads fall back to a `JsonRpcProvider` against `https://liteforge.rpc.caldera.xyz/http`, so the DApp stays useful in read-only mode with no injected wallet.
- No LitVM-proprietary precompiles are used — everything is plain EVM, so the contracts are portable and auditable by standard tooling.
- `foundry.toml` registers both `litvm_testnet` and a `litvm_mainnet` placeholder endpoint plus the Caldera explorer API for `forge verify-contract`.

---

## Design notes, trust model & known limits

**Intentionally minimal, not production DeFi**

- **Pseudo-randomness is not secure.** `HigherLowerArcade._draw` mixes `msg.sender`, `totalEntries` and the player's own score. A determined player can simulate the next card; there is no economic stake, so this is acceptable for a free-to-play testnet arcade. Do not attach value to it.
- **Hand-rolled ERC-20/ERC-721.** `HIToken`, `StreakBadgeNFT`, `GMMilestoneNFT` and `ArcadeLevelNFT` are deliberately small, dependency-free implementations rather than OpenZeppelin imports. Benefits: fully readable in one sitting, no transitive imports to verify on the explorer. Trade-offs: no `Permit`, no `_safeMint` receiver hooks, `safeTransferFrom` does not check `onERC721Received`.
- **XP weights are constants.** Changing them requires deploying a new `XPTracker` and re-pointing `HIToken` / `ArcadeLevelNFT`, both of which take the tracker in their constructors. This is deliberate — no upgradeability, no admin override of user XP.
- **`gmLog` grows forever.** `GMStreak.gmLog` appends every check-in for indexers; gas cost grows with history. Acceptable at testnet volumes; a production version would emit events only.
- **Top-10 leaderboard is O(n) to update** with a hard cap of 10 — bounded and cheap by construction.
- **The deployer is trusted only during deployment.** After `setGovernor`, the deployer has no special rights over the timelock or the governor. `HIToken`, `XPTracker`, `ArcadeGovernor` and `ArcadeTimelock` have no owner at all. `ArcadeLevelNFT` retains an owner for `setBaseURI` / `transferOwnership` — the owner should be transferred to the timelock for full decentralisation.

**Hardening checklist before any mainnet use**

1. Replace `_draw` with a commit–reveal or VRF-backed randomness source.
2. Route `ArcadeLevelNFT.setBaseURI` ownership into the timelock.
3. Adopt audited OZ ERC-20/ERC-721 (or add `_safeMint` receiver checks).
4. Replace the single-call governor payload with multi-call execution (`MAX_OPERATIONS` already allows for it).
5. Drop `gmLog` in favour of events, or paginate it.

---

## Roadmap

- Multi-call proposals (`MAX_OPERATIONS = 10` is already reserved)
- On-chain, paginated game history instead of client-side event scans
- Delegation UI that surfaces checkpoint history, not just current votes
- Commit–reveal randomness for a value-bearing game mode
- Move `ArcadeLevelNFT` ownership to the timelock once metadata is final

---

## License

MIT.
