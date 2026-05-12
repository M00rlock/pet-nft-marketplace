# Technical Design Document

## Overview

NFT Marketplace with Fractionalization — монорепозиторій з чотирма шарами: смарт-контракти (Hardhat + Solidity), Node.js індексер (TypeScript + PostgreSQL), React/Next.js фронтенд (Wagmi), та IPFS-зберігання метаданих. Деплой на zkSync Era Sepolia testnet.

---

## Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                        Browser / User                        │
└───────────────────────────┬─────────────────────────────────┘
                            │ HTTPS
┌───────────────────────────▼─────────────────────────────────┐
│              Frontend  (Next.js 14 / React 18)               │
│  Wagmi v2 · viem · RainbowKit · TanStack Query               │
│  Pages: /mint  /marketplace  /fractions  /dao                │
└──────┬──────────────────────────────────┬────────────────────┘
       │ REST (historical data)            │ JSON-RPC (live txs)
┌──────▼──────────────┐        ┌──────────▼──────────────────┐
│  Indexer (Node.js)  │        │   zkSync Era Sepolia RPC     │
│  Express · ethers   │        │   (Alchemy / public)         │
│  PostgreSQL         │        └──────────┬───────────────────┘
└─────────────────────┘                   │
                                ┌─────────▼───────────────────┐
                                │      Smart Contracts         │
                                │  NFTCollection               │
                                │  Marketplace                 │
                                │  FractionVault               │
                                │  FractionToken (per NFT)     │
                                │  NFTGovernor (DAO)           │
                                │  [Paymaster — optional]      │
                                └─────────────────────────────┘
                                          │
                                ┌─────────▼───────────────────┐
                                │   IPFS (Web3.Storage)        │
                                │   image CID + metadata CID   │
                                └─────────────────────────────┘
```

---

## Repository Structure

```
pet-nft-marketplace/
├── contracts/                  # Solidity source
│   ├── NFTCollection.sol
│   ├── Marketplace.sol
│   ├── FractionVault.sol
│   ├── FractionToken.sol
│   ├── NFTGovernor.sol
│   └── paymaster/
│       └── NFTPaymaster.sol    # optional
├── test/                       # Hardhat TypeScript tests
│   ├── NFTCollection.test.ts
│   ├── Marketplace.test.ts
│   ├── FractionVault.test.ts
│   ├── NFTGovernor.test.ts
│   └── integration/
│       ├── buyFlow.test.ts
│       ├── fractionalizeFlow.test.ts
│       └── daoFlow.test.ts
├── scripts/
│   ├── deploy.ts               # Hardhat deploy script
│   └── verify.ts
├── deployments/
│   └── zkSyncEraSepolia.json   # written by deploy script
├── indexer/                    # Node.js TypeScript service
│   ├── src/
│   │   ├── index.ts
│   │   ├── listener.ts
│   │   ├── api/
│   │   │   └── router.ts
│   │   ├── db/
│   │   │   ├── schema.sql
│   │   │   └── queries.ts
│   │   └── validators/
│   │       └── eventSchemas.ts
│   ├── package.json
│   └── .env.example
├── frontend/                   # Next.js 14 app
│   ├── app/
│   │   ├── layout.tsx
│   │   ├── page.tsx            # marketplace home
│   │   ├── mint/page.tsx
│   │   ├── fractions/page.tsx
│   │   └── dao/page.tsx
│   ├── components/
│   ├── hooks/
│   ├── lib/
│   │   ├── wagmi.ts
│   │   └── ipfs.ts
│   ├── package.json
│   └── .env.example
├── hardhat.config.ts
├── package.json
├── .solhint.json
├── .eslintrc.json
├── .env.example
├── README.md
└── LICENSE
```

---

## Smart Contracts

### 1. NFTCollection.sol

**Inherits:** `ERC721A`, `ERC2981`, `Ownable`

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

contract NFTCollection is ERC721A, ERC2981 {
    // tokenId → ipfs:// URI
    mapping(uint256 => string) private _tokenURIs;

    function mint(
        address to,
        uint256 quantity,       // 1–20
        string[] calldata uris, // length == quantity
        uint96 royaltyBps       // 0–10000
    ) external;

    function tokenURI(uint256 tokenId) public view override returns (string memory);
    function supportsInterface(bytes4) public view override(ERC721A, ERC2981) returns (bool);
}
```

**Key decisions:**
- Per-token royalty: `_setTokenRoyalty(tokenId, msg.sender, royaltyBps)` called for each minted token.
- URI stored in `_tokenURIs[tokenId]`; validated on-chain (`bytes(uri).length > 0` + prefix check via assembly for gas efficiency).
- No mint fee, no access control on `mint`.

---

### 2. Marketplace.sol

**Inherits:** `ReentrancyGuard`, `Pausable`, `Ownable`

```solidity
struct Listing {
    address seller;
    address tokenAddress;
    uint256 tokenId;
    uint256 price;      // wei, ETH only
}

mapping(address => mapping(uint256 => Listing)) public listings;
// key: tokenAddress → tokenId → Listing

function listNFT(address tokenAddress, uint256 tokenId, uint256 price) external;
function cancelListing(address tokenAddress, uint256 tokenId) external;
function buyNFT(address tokenAddress, uint256 tokenId) external payable nonReentrant;
```

**Buy flow (atomic):**
1. Load listing, validate `msg.value == listing.price`.
2. Delete listing (CEI pattern — state change before transfer).
3. Call `IERC2981.royaltyInfo(tokenId, price)` → `(recipient, royaltyAmount)`.
4. Revert if `royaltyAmount >= price`.
5. `IERC721(tokenAddress).safeTransferFrom(seller, buyer, tokenId)`.
6. `payable(recipient).transfer(royaltyAmount)`.
7. `payable(seller).transfer(price - royaltyAmount)`.
8. Emit `Sold`.

**Stale listing:** `buyNFT` will revert at step 5 (NFT no longer owned by seller). Seller can still call `cancelListing`.

---

### 3. FractionVault.sol + FractionToken.sol

**FractionVault inherits:** `ReentrancyGuard`, `IERC721Receiver`

```solidity
struct VaultEntry {
    address nftAddress;
    uint256 tokenId;
    address depositor;
    address fractionToken;  // deployed FractionToken address
    bool active;
}

// nftAddress → tokenId → VaultEntry
mapping(address => mapping(uint256 => VaultEntry)) public vaults;

function deposit(address nftAddress, uint256 tokenId) external;
function redeem(address nftAddress, uint256 tokenId) external nonReentrant;
function onERC721Received(...) external pure returns (bytes4);
```

**Deposit flow:**
1. Validate no active vault for `(nftAddress, tokenId)`.
2. `IERC721(nftAddress).safeTransferFrom(msg.sender, address(this), tokenId)` — triggers `onERC721Received`.
3. Deploy new `FractionToken(name, symbol, msg.sender, 1_000_000 * 1e18)`.
4. Store `VaultEntry{..., active: true}`.
5. Emit `Deposited(tokenId, msg.sender, fractionTokenAddress, totalSupply)`.

**Redeem flow:**
1. Load vault entry, validate `active == true`.
2. Load `FractionToken` at stored address.
3. Validate `fractionToken.balanceOf(msg.sender) == fractionToken.totalSupply()`.
4. `fractionToken.burnAll(msg.sender)` — burns entire supply.
5. Mark `vault.active = false` (allows re-deposit).
6. `IERC721(nftAddress).safeTransferFrom(address(this), msg.sender, tokenId)`.
7. Emit `Redeemed`.

**FractionToken.sol** — minimal ERC-20:
```solidity
contract FractionToken is ERC20, ERC20Votes {
    address public immutable vault;

    constructor(string memory name, string memory symbol,
                address recipient, uint256 supply, address _vault) {
        vault = _vault;
        _mint(recipient, supply);
    }

    // Only vault can burn (called during redeem)
    function burnAll(address from) external {
        require(msg.sender == vault, "only vault");
        _burn(from, totalSupply());
    }
}
```

`ERC20Votes` extension is required for OpenZeppelin Governor snapshot voting.

---

### 4. NFTGovernor.sol

**Inherits:** `Governor`, `GovernorSettings`, `GovernorCountingSimple`, `GovernorVotes`, `GovernorVotesQuorumFraction`

```solidity
constructor(IVotes _token, uint256 _quorumNumerator)
    Governor("NFTGovernor")
    GovernorSettings(1, 50400, 0)          // delay, period, proposalThreshold
    GovernorVotes(_token)
    GovernorVotesQuorumFraction(_quorumNumerator)  // e.g. 1 = 1%
{}
```

**Key decisions:**
- `proposalThreshold` = 0 in `GovernorSettings` — threshold enforced manually via `_getVotes` check at 1% in `propose` override, OR use `GovernorSettings` with threshold set to `totalSupply / 100` (updated at deploy time).
- `quorumNumerator` = 1 (1%), configurable at deploy.
- No Timelock for MVP — proposals execute immediately after `Succeeded`.
- `FractionToken` must implement `ERC20Votes` (includes `delegate` — users must self-delegate to activate voting power).

**Frontend note:** Users need to call `fractionToken.delegate(userAddress)` once before voting. Frontend must prompt this.

---

### 5. NFTPaymaster.sol (optional)

**Inherits:** `BasePaymaster` (from `@account-abstraction/contracts`)

```solidity
mapping(bytes4 => bool) public whitelistedSelectors;

function validatePaymasterUserOp(
    UserOperation calldata userOp,
    bytes32,
    uint256 maxCost
) external override returns (bytes memory context, uint256 validationData) {
    bytes4 selector = bytes4(userOp.callData[:4]);
    require(whitelistedSelectors[selector], "selector not whitelisted");
    require(getDeposit() >= maxCost, "insufficient deposit");
    return ("", 0);
}
```

Whitelisted selectors: `mint`, `listNFT`, `buyNFT`, `deposit`, `redeem`, `castVote`.

---

## Indexer (Node.js / TypeScript)

### Stack
- **Runtime:** Node.js 20 LTS
- **Framework:** Express 4
- **Blockchain:** ethers.js v6
- **DB:** PostgreSQL 15 via `pg` + raw SQL (no ORM for simplicity)
- **Validation:** zod

### Database Schema

```sql
-- migrations/001_init.sql

CREATE TABLE indexed_state (
  key   TEXT PRIMARY KEY,
  value TEXT NOT NULL
);
-- key='lastBlock', value='12345678'

CREATE TABLE listings (
  id            SERIAL PRIMARY KEY,
  token_address TEXT NOT NULL,
  token_id      TEXT NOT NULL,
  seller        TEXT NOT NULL,
  price_wei     TEXT NOT NULL,
  status        TEXT NOT NULL DEFAULT 'active', -- active | sold | cancelled
  listed_at     BIGINT NOT NULL,   -- block number
  updated_at    BIGINT,
  UNIQUE(token_address, token_id, status)
);

CREATE TABLE fractions (
  id                   SERIAL PRIMARY KEY,
  token_address        TEXT NOT NULL,
  token_id             TEXT NOT NULL,
  depositor            TEXT NOT NULL,
  fraction_token_addr  TEXT NOT NULL,
  total_supply         TEXT NOT NULL,
  status               TEXT NOT NULL DEFAULT 'active', -- active | redeemed
  deposited_at         BIGINT NOT NULL,
  redeemed_at          BIGINT,
  UNIQUE(token_address, token_id)
);

CREATE TABLE proposals (
  id              SERIAL PRIMARY KEY,
  proposal_id     TEXT NOT NULL UNIQUE,
  proposer        TEXT NOT NULL,
  description     TEXT NOT NULL,
  targets         JSONB NOT NULL,
  values          JSONB NOT NULL,
  calldatas       JSONB NOT NULL,
  state           TEXT NOT NULL DEFAULT 'Pending',
  votes_for       TEXT NOT NULL DEFAULT '0',
  votes_against   TEXT NOT NULL DEFAULT '0',
  created_at      BIGINT NOT NULL,
  executed_at     BIGINT
);

CREATE TABLE events (
  id          SERIAL PRIMARY KEY,
  event_name  TEXT NOT NULL,
  block_number BIGINT NOT NULL,
  tx_hash     TEXT NOT NULL,
  payload     JSONB NOT NULL,
  created_at  TIMESTAMPTZ DEFAULT NOW()
);
CREATE INDEX ON events(event_name);
CREATE INDEX ON events(block_number);
```

### Event Listener Architecture

```
listener.ts
  └── for each contract ABI:
        contract.on(eventName, handler)
        handler:
          1. validate payload (zod schema)
          2. upsert into relevant table
          3. update indexed_state.lastBlock
          4. insert into events table
```

Reconnect logic: `provider.on('error', ...)` → exponential backoff (1s, 2s, 4s, 8s, 16s, then give up + log).

### REST API Endpoints

| Method | Path | Description |
|--------|------|-------------|
| GET | `/listings?limit=&offset=` | Active listings |
| GET | `/listings/:tokenAddress/:tokenId` | Single listing |
| GET | `/fractions?limit=&offset=` | Active vaults |
| GET | `/fractions/:tokenAddress/:tokenId` | Single vault |
| GET | `/proposals?limit=&offset=` | All proposals |
| GET | `/proposals/:proposalId` | Single proposal with vote counts |
| GET | `/events?limit=&offset=&event=` | Paginated event log |

All paginated endpoints default to `limit=20`, max `limit=100`.

---

## Frontend (Next.js 14 / React 18)

### Stack
- **Next.js 14** (App Router)
- **Wagmi v2** + **viem** — contract reads/writes
- **RainbowKit** — wallet connection UI (MetaMask, WalletConnect, Coinbase)
- **TanStack Query v5** — server state, caching
- **Tailwind CSS** — styling
- **Web3.Storage** client — IPFS uploads

### Wagmi Config (`lib/wagmi.ts`)

```typescript
import { createConfig, http } from 'wagmi'
import { zkSyncSepoliaTestnet } from 'wagmi/chains'
import { injected, walletConnect, coinbaseWallet } from 'wagmi/connectors'

export const config = createConfig({
  chains: [zkSyncSepoliaTestnet],
  connectors: [
    injected(),
    walletConnect({ projectId: process.env.NEXT_PUBLIC_WC_PROJECT_ID! }),
    coinbaseWallet({ appName: 'NFT Marketplace' }),
  ],
  transports: { [zkSyncSepoliaTestnet.id]: http(process.env.NEXT_PUBLIC_RPC_URL) },
})
```

### Page → Component Map

```
app/
├── page.tsx                    → <ListingsGrid> (reads from Indexer API)
│                                 <ListingCard> (buy button)
├── mint/page.tsx               → <MintForm>
│                                   <ImageUploader> (→ IPFS)
│                                   <MetadataForm>
│                                   <MintButton> (useWriteContract)
├── fractions/page.tsx          → <MyNFTs> (owned tokens)
│                                 <FractionalizeForm> (deposit)
│                                 <FractionHoldings> (ERC-20 balances)
│                                 <RedeemButton>
└── dao/page.tsx                → <ProposalList> (reads from Indexer)
                                  <ProposalCard> (vote buttons)
                                  <CreateProposalForm>
                                  <DelegateButton> (ERC20Votes.delegate)
```

### Key Hooks

```typescript
// hooks/useNFTCollection.ts
export function useMint() {
  return useWriteContract({ abi: NFTCollectionABI, functionName: 'mint' })
}

// hooks/useMarketplace.ts
export function useListNFT() { ... }
export function useBuyNFT() { ... }
export function useCancelListing() { ... }

// hooks/useFractionVault.ts
export function useDeposit() { ... }
export function useRedeem() { ... }

// hooks/useDAO.ts
export function usePropose() { ... }
export function useCastVote() { ... }
export function useDelegate() { ... }

// hooks/useContractEvents.ts
// Subscribes to Listed/Sold/Deposited/Redeemed events
// Invalidates TanStack Query cache on event → triggers re-fetch within 5s
```

### IPFS Upload Flow (`lib/ipfs.ts`)

```typescript
import { Web3Storage } from 'web3.storage'

export async function uploadToIPFS(image: File, metadata: NFTMetadata): Promise<string> {
  // 1. Upload image → get image CID
  // 2. Build metadata JSON with image: `ipfs://{imageCID}`
  // 3. Upload metadata JSON → get metadata CID
  // 4. Return `ipfs://{metadataCID}`
}
```

---

## Deployment

### Target Network: zkSync Era Sepolia Testnet
- Chain ID: 300
- RPC: `https://sepolia.era.zksync.dev`
- Explorer: `https://sepolia.explorer.zksync.io`

### Deploy Script (`scripts/deploy.ts`)

```
Deploy order:
1. NFTCollection
2. Marketplace
3. FractionToken (implementation — deployed by FractionVault at runtime, no pre-deploy needed)
4. FractionVault
5. NFTGovernor(fractionToken.address, quorumNumerator=1)
   ⚠ Governor needs a token address — for MVP, deploy with a placeholder FractionToken
   or deploy Governor per-vault (out of scope). For MVP: one shared Governor
   pointing to the first FractionToken deployed, or use a GovernorFactory pattern.
   Decision: deploy a "demo" FractionToken at deploy time for Governor; real vaults
   use their own FractionTokens for trading but DAO uses the demo token for portfolio demo.
6. [NFTPaymaster — optional]
7. Write deployments/zkSyncEraSepolia.json
```

**Note on Governor + multiple FractionTokens:** The DAO requirement is for portfolio demo purposes. For MVP, the Governor is deployed with a single designated `FractionToken` (the first vault's token or a dedicated governance token). This is documented in README as a known simplification.

### Environment Variables

**Root `.env.example`:**
```
PRIVATE_KEY=0x...                    # deployer wallet private key
ZKSYNC_RPC_URL=https://sepolia.era.zksync.dev
ETHERSCAN_API_KEY=...                # for contract verification
```

**`indexer/.env.example`:**
```
DATABASE_URL=postgresql://user:pass@localhost:5432/nft_marketplace
RPC_URL=https://sepolia.era.zksync.dev
NFT_CONTRACT_ADDRESS=0x...
MARKETPLACE_CONTRACT_ADDRESS=0x...
VAULT_CONTRACT_ADDRESS=0x...
GOVERNOR_CONTRACT_ADDRESS=0x...
DEPLOYMENT_BLOCK=0                   # block to start indexing from
PORT=3001
FRONTEND_ORIGIN=http://localhost:3000
```

**`frontend/.env.example`:**
```
NEXT_PUBLIC_RPC_URL=https://sepolia.era.zksync.dev
NEXT_PUBLIC_WC_PROJECT_ID=...
NEXT_PUBLIC_NFT_CONTRACT_ADDRESS=0x...
NEXT_PUBLIC_MARKETPLACE_CONTRACT_ADDRESS=0x...
NEXT_PUBLIC_VAULT_CONTRACT_ADDRESS=0x...
NEXT_PUBLIC_GOVERNOR_CONTRACT_ADDRESS=0x...
NEXT_PUBLIC_INDEXER_URL=http://localhost:3001
NEXT_PUBLIC_WEB3_STORAGE_TOKEN=...
```

---

## CI/CD (GitHub Actions)

### `.github/workflows/ci.yml` — runs on every PR

```yaml
jobs:
  lint-solidity:    # solhint contracts/**/*.sol
  lint-typescript:  # eslint indexer/src frontend/app frontend/components
  test:             # npx hardhat test --network hardhat
  coverage:         # npx hardhat coverage → report to PR comment
```

### `.github/workflows/cd.yml` — runs on push to `main`

```yaml
jobs:
  deploy:
    needs: [lint-solidity, lint-typescript, test]
    steps:
      - npx hardhat run scripts/deploy.ts --network zkSyncEraSepolia
      - git add deployments/zkSyncEraSepolia.json
      - git commit -m "chore: update deployment addresses [skip ci]"
      - git push
```

---

## Security Considerations

| Risk | Mitigation |
|------|-----------|
| Reentrancy in `buyNFT` | CEI pattern + `ReentrancyGuard` |
| Reentrancy in `redeem` | CEI pattern + `ReentrancyGuard` |
| Royalty overflow | Revert if `royaltyAmount >= price` |
| Flash loan governance attack | `ERC20Votes` uses past-block snapshot |
| Stale listings | Documented behaviour; `buyNFT` reverts naturally |
| Vault double-deposit | `active` flag check in `deposit` |
| Integer overflow | Solidity 0.8.20 built-in checks |
| IPFS URI spoofing | On-chain `ipfs://` prefix validation |

---

## Open Questions / Known Simplifications

1. **Governor token scope:** MVP uses one FractionToken per Governor. Multi-vault DAO governance is out of scope.
2. **ERC20Votes delegation:** Users must call `delegate(self)` before voting. Frontend prompts this but it's an extra UX step.
3. **zkSync Era compatibility:** ERC-721A and OpenZeppelin v5 are compatible with zkSync Era. Paymaster uses zkSync-native AA, not ERC-4337 EntryPoint (zkSync has its own AA model). The Paymaster section in requirements maps to zkSync's `IPaymaster` interface, not EIP-4337 EntryPoint.
4. **No AMM for fraction trading:** Fraction_Tokens are standard ERC-20 — users transfer them manually or list them on the Marketplace. No DEX integration in MVP.
