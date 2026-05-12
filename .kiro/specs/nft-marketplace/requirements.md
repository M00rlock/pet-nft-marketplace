# Requirements Document

## Introduction

NFT Marketplace with Fractionalization — open-source портфоліо-проєкт (MIT), що реалізує повний цикл роботи з NFT: мінтинг, торгівля, фракціоналізація, DAO-голосування та опціональна підтримка account abstraction (ERC-4337). Система розгортається на zkEVM тестнеті (zkSync Era / Linea / Scroll) і включає смарт-контракти, React/Next.js фронтенд, Node.js індексер та IPFS-зберігання метаданих.

## Glossary

- **NFT_Contract**: Смарт-контракт стандарту ERC-721A (або ERC-721) для мінтингу та управління NFT-токенами.
- **Marketplace_Contract**: Смарт-контракт для лістингу, купівлі та скасування продажу NFT.
- **Vault_Contract**: Смарт-контракт фракціоналізації — приймає NFT на депозит і випускає ERC-20 частки.
- **Fraction_Token**: ERC-20 токен, що представляє часткову власність на NFT у Vault_Contract.
- **DAO_Contract**: Смарт-контракт on-chain голосування на базі OpenZeppelin Governor або мінімальної реалізації.
- **Paymaster**: Опціональний ERC-4337 контракт, що дозволяє спонсорувати газ для транзакцій користувачів.
- **Indexer**: Node.js/TypeScript сервіс, що читає події блокчейну та зберігає їх у PostgreSQL.
- **Frontend**: React/Next.js застосунок для взаємодії користувача з усіма контрактами.
- **IPFS_Storage**: Децентралізоване сховище метаданих NFT (Web3.Storage або nft.storage).
- **Royalty_Recipient**: Адреса, що отримує роялті відповідно до EIP-2981.
- **Seller**: Власник NFT, що виставляє токен на продаж.
- **Buyer**: Користувач, що купує NFT на маркетплейсі.
- **Fraction_Holder**: Власник Fraction_Token, що має право голосу в DAO та може брати участь у redeem.
- **Creator**: Адреса, що здійснила мінтинг NFT і є первинним Royalty_Recipient.
- **zkEVM_Network**: Тестова мережа з EVM-сумісністю на базі ZK-rollup (zkSync Era, Linea або Scroll).

---

## Requirements

### Requirement 1: NFT Мінтинг

**User Story:** As a Creator, I want to mint NFTs with on-chain metadata references and royalty settings, so that I can publish digital assets and receive royalties on secondary sales.

#### Acceptance Criteria

1. THE NFT_Contract SHALL implement the ERC-721A standard to enable gas-efficient batch minting.
2. WHEN a Creator calls the mint function with a non-empty metadata URI starting with `ipfs://` and royalty basis points in the range 0–10000, THE NFT_Contract SHALL mint the token, assign ownership to the Creator, set the Royalty_Recipient to the Creator's address, and store the tokenURI.
3. WHEN a royalty query is made for a token (EIP-2981 `royaltyInfo`), THE NFT_Contract SHALL return the Royalty_Recipient address and the royalty amount calculated as `salePrice * royaltyBps / 10000`.
4. IF a mint is called with royalty basis points greater than 10000, THEN THE NFT_Contract SHALL revert the transaction with a descriptive error.
5. IF a mint is called with an empty metadata URI or a URI that does not start with `ipfs://`, THEN THE NFT_Contract SHALL revert the transaction with a descriptive error.
6. THE NFT_Contract SHALL expose a `tokenURI(tokenId)` function that returns the IPFS URI stored at mint time.
7. WHEN a batch mint is requested for a quantity between 1 and 20 tokens in a single transaction, THE NFT_Contract SHALL mint all tokens and assign sequential token IDs.
8. IF a batch mint is requested with a quantity of 0 or greater than 20, THEN THE NFT_Contract SHALL revert the transaction with a descriptive error.
9. IF `tokenURI` is called with a non-existent tokenId, THEN THE NFT_Contract SHALL revert the transaction with a descriptive error.
10. THE NFT_Contract mint function SHALL be public and require no fee (permissionless minting for portfolio demo); there is no onlyOwner restriction on minting.

---

### Requirement 2: IPFS Metadata Storage

**User Story:** As a Creator, I want NFT metadata stored on IPFS, so that the asset data is decentralized and persistent.

#### Acceptance Criteria

1. WHEN a Creator uploads an image (≤ 50 MB) and metadata JSON via the Frontend, THE IPFS_Storage SHALL return a content-addressed CID within 30 seconds.
2. THE IPFS_Storage SHALL store metadata in the ERC-721 Metadata JSON Schema format with fields: `name`, `description`, `image`, and optional `attributes`.
3. WHEN the IPFS_Storage returns a CID, THE Frontend SHALL construct the tokenURI as `ipfs://{CID}` before calling the mint function.
4. IF the IPFS_Storage upload fails, THEN THE Frontend SHALL display an error message containing the failure reason and SHALL NOT proceed with the mint transaction.
5. IF the metadata JSON is missing any required field (`name`, `description`, or `image`), THEN THE Frontend SHALL display a validation error identifying the missing field and SHALL NOT call the IPFS_Storage upload.
6. WHEN a Creator uploads the same metadata JSON twice, THE IPFS_Storage SHALL return the same CID both times (content-addressing round-trip property).

---

### Requirement 3: Marketplace — Лістинг та Купівля

**User Story:** As a Seller, I want to list my NFT at a fixed price and as a Buyer I want to purchase it, so that NFTs can be traded on the marketplace.

#### Acceptance Criteria

1. WHEN a Seller calls `listNFT(tokenId, price)` with a price greater than 0 and has approved the Marketplace_Contract as operator (via `approve` or `setApprovalForAll`), THE Marketplace_Contract SHALL create a listing record with the token ID, Seller address, and price. The listing currency SHALL be native ETH only; ERC-20 payment tokens are out of scope for MVP.
2. WHEN a Buyer calls `buyNFT(tokenId)` and sends the exact listing price in ETH/native token, THE Marketplace_Contract SHALL transfer the NFT to the Buyer, distribute funds to the Seller and Royalty_Recipient, and remove the listing.
3. WHEN a purchase is executed for an active listing, THE Marketplace_Contract SHALL calculate the royalty amount using EIP-2981 `royaltyInfo`; IF the royalty amount returned is greater than or equal to the listing price, THEN THE Marketplace_Contract SHALL revert the transaction; OTHERWISE THE Marketplace_Contract SHALL transfer the royalty amount to the Royalty_Recipient and transfer the remainder to the Seller.
4. WHEN a Seller calls `cancelListing(tokenId)` for an active listing they own, THE Marketplace_Contract SHALL remove the listing and emit a `ListingCancelled` event.
5. IF a Buyer calls `buyNFT(tokenId)` with an incorrect ETH amount, THEN THE Marketplace_Contract SHALL revert the transaction with a descriptive error.
6. IF a Seller calls `listNFT` for a token they do not own, THEN THE Marketplace_Contract SHALL revert the transaction.
7. IF a Seller calls `listNFT` without prior ERC-721 approval of the Marketplace_Contract, THEN THE Marketplace_Contract SHALL revert the transaction.
8. WHEN a listing is created, THE Marketplace_Contract SHALL emit a `Listed(tokenId, seller, price)` event.
9. WHEN a purchase is completed, THE Marketplace_Contract SHALL emit a `Sold(tokenId, buyer, price)` event.
10. IF `buyNFT` is called for a tokenId that has no active listing, THEN THE Marketplace_Contract SHALL revert the transaction with a descriptive error.
11. IF `listNFT` is called for a tokenId that already has an active listing, THEN THE Marketplace_Contract SHALL revert the transaction with a descriptive error.
12. IF a caller other than the listing's Seller calls `cancelListing(tokenId)`, THEN THE Marketplace_Contract SHALL revert the transaction with a descriptive error.
13. IF an NFT is transferred directly (outside the Marketplace_Contract) while an active listing exists for that tokenId, THE Marketplace_Contract SHALL allow the listing to remain in storage but `buyNFT` SHALL revert because the NFT transfer will fail; the original Seller or any caller MAY call `cancelListing` to clean up the stale listing (no ownership check on cancel for stale listings is not required — the Seller address stored in the listing is the only authorized canceller).

---

### Requirement 4: Fractionalization Vault

**User Story:** As an NFT owner, I want to deposit my NFT into a vault and receive ERC-20 fraction tokens, so that I can sell partial ownership of the asset.

#### Acceptance Criteria

1. WHEN an owner calls `deposit(tokenId)` on the Vault_Contract and has approved the Vault_Contract as operator, THE Vault_Contract SHALL transfer the NFT from the owner, mint exactly 1,000,000 × 10^18 Fraction_Tokens to the depositor, and lock the NFT; IF the NFT transfer fails due to missing approval, THEN THE Vault_Contract SHALL revert with a descriptive error.
2. THE Vault_Contract SHALL use a factory pattern: each `deposit(tokenId)` call deploys (or initialises) a dedicated FractionToken ERC-20 contract for that tokenId, so that each NFT has its own independent Fraction_Token supply and `redeem()` call. The Vault_Contract tracks a mapping of `tokenId → FractionToken address`.
3. THE Vault_Contract SHALL support exactly one active vault per NFT tokenId at any time; IF `deposit(tokenId)` is called for a tokenId that is already locked in the Vault_Contract, THEN THE Vault_Contract SHALL revert with a descriptive error.
4. WHEN a Fraction_Holder calls `redeem(tokenId)` and holds 100% of the total Fraction_Token supply for that vault (i.e., balance equals 1,000,000 × 10^18), THE Vault_Contract SHALL burn all Fraction_Tokens held by the caller and transfer the NFT back to the caller. After a successful redeem the vault entry for that tokenId SHALL be cleared, allowing the same NFT to be deposited again in the future.
5. IF a Fraction_Holder calls `redeem(tokenId)` while holding less than 100% of the Fraction_Token supply, THEN THE Vault_Contract SHALL revert the transaction with a descriptive error.
6. WHILE an NFT is locked in the Vault_Contract, THE Vault_Contract SHALL NOT expose any function that transfers the NFT to an address other than the original depositor via the `redeem` path; all other NFT transfer paths from the Vault_Contract are prohibited by design.
7. THE FractionToken contract SHALL implement the ERC-20 standard (including `transfer`, `approve`, `transferFrom`) to enable trading on DEXes and the Marketplace_Contract.
8. WHEN a deposit is made, THE Vault_Contract SHALL emit a `Deposited(tokenId, depositor, fractionTokenAddress, totalSupply)` event.
9. WHEN a redeem is executed, THE Vault_Contract SHALL emit a `Redeemed(tokenId, redeemer)` event.

---

### Requirement 5: DAO Голосування

**User Story:** As a Fraction_Holder, I want to participate in on-chain governance votes, so that collective decisions about the fractionalized NFT can be made transparently.

#### Acceptance Criteria

1. THE DAO_Contract SHALL implement on-chain proposal creation, voting, and execution using OpenZeppelin Governor or a minimal equivalent.
2. WHEN a Fraction_Holder whose Fraction_Token balance at `block.number - 1` is ≥ 1% of the total Fraction_Token supply calls `propose(targets, values, calldatas, description)`, THE DAO_Contract SHALL create a proposal and emit a `ProposalCreated(proposalId, proposer, targets, values, calldatas, description)` event.
3. WHEN a Fraction_Holder calls `castVote(proposalId, support)` during the active voting period, THE DAO_Contract SHALL record the vote weighted by the Fraction_Holder's Fraction_Token balance at the proposal snapshot block; IF the same address calls `castVote` for the same proposalId a second time, THEN THE DAO_Contract SHALL revert the transaction.
4. WHEN the voting period for a proposal ends, IF the total votes in favour exceed votes against AND the total votes in favour represent ≥ 1% of the total Fraction_Token supply at the snapshot block, THEN THE DAO_Contract SHALL mark the proposal as `Succeeded`. The quorum is set to 1% to make the portfolio demo operable with a small number of token holders; it SHALL be configurable as a constructor parameter at deployment time.
5. WHEN a `Succeeded` proposal is executed via `execute(targets, values, calldatas, descriptionHash)` with parameters matching the original `propose` call, THE DAO_Contract SHALL call the specified target contracts with the provided calldata. Any address MAY call `execute` on a `Succeeded` proposal.
6. IF a Fraction_Holder calls `castVote` after the voting period has ended, THEN THE DAO_Contract SHALL revert the transaction regardless of the caller's voting power.
7. IF `execute` is called on a proposal that is not in the `Succeeded` state, THEN THE DAO_Contract SHALL revert the transaction with a descriptive error.
8. THE DAO_Contract SHALL use Fraction_Token balances at the proposal snapshot block as voting power (1 Fraction_Token = 1 vote).
9. THE DAO_Contract SHALL enforce a minimum voting delay of 1 block and a minimum voting period of 50400 blocks (~7 days at 12s/block).

---

### Requirement 6: Опціональний ERC-4337 Paymaster

**User Story:** As a user without ETH for gas, I want transactions to be sponsored by a Paymaster, so that I can interact with the marketplace without holding native tokens.

#### Acceptance Criteria

1. WHERE the ERC-4337 feature is enabled, THE Paymaster SHALL implement the `IPaymaster` interface from the ERC-4337 specification.
2. WHERE the ERC-4337 feature is enabled, WHEN a UserOperation is submitted for a whitelisted contract function AND the Paymaster's deposit in the EntryPoint is sufficient to cover the operation's gas cost, THE Paymaster SHALL validate and sponsor the gas cost.
3. WHERE the ERC-4337 feature is enabled, IF a UserOperation targets a non-whitelisted function, THEN THE Paymaster SHALL revert with an error indicating the function is not eligible for sponsorship.
4. WHERE the ERC-4337 feature is enabled, THE Paymaster SHALL maintain a deposit in the EntryPoint contract of at least `100 × estimatedGasPerOp × currentBaseFee` wei, where `estimatedGasPerOp` is the median gas used by the last 10 sponsored operations; IF fewer than 10 operations have been sponsored, THE Paymaster SHALL use a hardcoded fallback value of 300,000 gas as `estimatedGasPerOp`.
5. WHERE the ERC-4337 feature is enabled, IF the Paymaster's deposit in the EntryPoint falls below `10 × estimatedGasPerOp × currentBaseFee` wei, THEN THE Paymaster SHALL reject all sponsorship requests until the deposit is replenished above the minimum threshold.

---

### Requirement 7: Frontend — Wallet та UI Flows

**User Story:** As a user, I want a web interface to connect my wallet and perform all marketplace actions, so that I can interact with the contracts without writing code.

#### Acceptance Criteria

1. THE Frontend SHALL support wallet connection via Wagmi with MetaMask, WalletConnect, and Coinbase Wallet providers.
2. WHEN a user connects a wallet, THE Frontend SHALL display the connected address, native token balance, and current network.
3. IF a user is connected to an unsupported network, THEN THE Frontend SHALL display a banner prompting the user to switch to the configured zkEVM_Network; the user SHALL be able to dismiss the banner and remain on the unsupported network.
4. WHEN a user completes the Mint flow (upload image → upload metadata to IPFS → call `mint`), THE Frontend SHALL display the transaction hash and new token ID upon transaction confirmation.
5. WHEN a user completes the List flow (select owned NFT → enter price → approve Marketplace_Contract → call `listNFT`), THE Frontend SHALL display a confirmation message upon transaction confirmation.
6. WHEN a user completes the Buy flow (browse listings → select listing → call `buyNFT` with correct ETH value), THE Frontend SHALL display a confirmation message upon transaction confirmation.
7. WHEN a user completes the Fractionalize flow (select owned NFT → approve Vault_Contract → call `deposit`), THE Frontend SHALL display the resulting Fraction_Token balance upon transaction confirmation.
8. WHEN a user interacts with the DAO flow (view proposals → cast vote → create proposal if threshold met), THE Frontend SHALL display the current proposal state as one of: `Pending`, `Active`, `Succeeded`, `Defeated`, or `Executed` upon transaction confirmation.
9. IF a transaction is rejected by the user in their wallet OR fails on-chain, THEN THE Frontend SHALL display an error message containing the rejection reason or on-chain revert message.
10. WHEN a `Listed`, `Sold`, `ListingCancelled`, `Deposited`, or `Redeemed` event is emitted on-chain, THE Frontend SHALL update the relevant UI component within 5 seconds by subscribing to contract events via Wagmi's `useContractEvent` or equivalent.

---

### Requirement 8: Indexer та Backend

**User Story:** As a developer, I want a backend indexer that tracks on-chain events, so that the Frontend can query historical data efficiently without relying solely on RPC calls.

#### Acceptance Criteria

1. WHEN a `Listed`, `Sold`, `ListingCancelled`, `Deposited`, `Redeemed`, `ProposalCreated`, `VoteCast`, or `ProposalExecuted` event is emitted on zkEVM_Network, THE Indexer SHALL persist the event data to PostgreSQL within 10 seconds; IF processing exceeds 10 seconds, THE Indexer SHALL log a warning with the block number and event type and continue processing.
2. THE Indexer SHALL expose a REST API with endpoints: `GET /listings` (active listings, paginated with `limit` ≤ 100 and `offset`), `GET /listings/:tokenId`, `GET /fractions/:vaultId`, `GET /events` (paginated event history with `limit` ≤ 100 and `offset`), `GET /proposals` (paginated DAO proposals), `GET /proposals/:proposalId` (single proposal with vote counts).
3. IF the Indexer loses connection to the RPC node, THEN THE Indexer SHALL attempt reconnection with exponential backoff (max 5 retries, starting at 1 second, doubling each attempt) and log each attempt with the retry count and delay.
4. WHEN the Indexer starts, IF a last-indexed block number exists in PostgreSQL, THEN THE Indexer SHALL resume processing from that block number; IF no last-indexed block exists, THEN THE Indexer SHALL start from the contract deployment block number specified in the configuration.
5. WHEN an event is received from the RPC node, THE Indexer SHALL validate that the event data matches the expected on-chain event schema (correct field names, types, and non-null required fields); IF validation passes, THE Indexer SHALL write the event to PostgreSQL; IF validation fails, THE Indexer SHALL skip the event and log a warning with the raw event data.
6. THE Indexer REST API SHALL include CORS headers permitting requests from the configured Frontend origin; no authentication is required for MVP.

---

### Requirement 9: Тести

**User Story:** As a developer, I want comprehensive automated tests, so that contract correctness is verified before deployment.

#### Acceptance Criteria

1. THE NFT_Contract, Marketplace_Contract, Vault_Contract, and DAO_Contract SHALL each have unit tests covering all public functions and revert conditions using **Hardhat** as the single consistently-used test framework with TypeScript test files.
2. THE test suite SHALL achieve at least 90% line coverage AND 90% branch coverage for each contract when all unit tests are executed.
3. THE test suite SHALL include an integration test for the mint → list → buy flow that asserts: NFT ownership transferred to Buyer, Seller received `price − royalty` ETH, and Royalty_Recipient received `royalty` ETH.
4. THE test suite SHALL include an integration test for the mint → deposit → fractionalize → redeem flow that asserts: NFT locked in Vault after deposit, Fraction_Token supply equals 1,000,000 × 10^18, and NFT returned to redeemer after burning 100% of supply.
5. THE test suite SHALL include an integration test for the mint → deposit → DAO propose → vote → execute flow that asserts: proposal created with correct parameters, votes recorded at snapshot block, and proposal marked `Succeeded` after quorum and majority are reached.
6. WHEN `tokenURI(id)` is called after minting a token with a given URI, THE NFT_Contract SHALL return the exact URI provided at mint time.
7. WHEN a listing is created with a given price and seller, querying the listing SHALL return the same price and seller address.
8. WHEN Fraction_Tokens are distributed across multiple holders after a vault deposit, the sum of all holder balances SHALL equal 1,000,000 × 10^18.

---

### Requirement 10: CI/CD та Деплой

**User Story:** As a developer, I want automated CI/CD pipelines, so that code quality is enforced and deployments are reproducible.

#### Acceptance Criteria

1. THE GitHub Actions CI pipeline SHALL run on every pull request: compile contracts, run all tests, and report coverage.
2. WHEN all CI checks pass on the `main` branch, THE GitHub Actions CD pipeline SHALL deploy contracts to the configured zkEVM_Network testnet using a deployment script.
3. WHEN the CD pipeline is triggered and all CI checks pass on the `main` branch, THE deployment script SHALL write deployed contract addresses to a `deployments/{network}.json` file and commit it to the repository.
4. IF any CI test fails, THEN THE GitHub Actions pipeline SHALL block the merge and report the failing test names.
5. THE CI pipeline SHALL run Solidity linting (Solhint) and TypeScript linting (ESLint) as separate, independently-reported steps.
6. IF the Solhint or ESLint step reports any error-level finding, THEN THE CI pipeline SHALL fail that step and block the merge.

---

### Requirement 11: Документація та Ліцензія

**User Story:** As a developer using Kiro IDE, I want clear README documentation and MIT licensing, so that I can set up and run the project locally.

#### Acceptance Criteria

1. THE repository SHALL contain a `README.md` with the following sections: Project Overview, Architecture Diagram (showing Frontend, Backend/Indexer, Smart Contracts, IPFS, and zkEVM_Network and their interactions), Prerequisites, Local Setup, Running Tests, Deploying to zkEVM_Network, and Frontend Usage.
2. THE README.md SHALL include a "Kiro IDE" section with the following steps: (a) open the repository folder in Kiro IDE, (b) install dependencies via the integrated terminal, (c) run tests using the configured test command, and (d) start the local development node.
3. THE repository SHALL contain a `LICENSE` file with the MIT license text.
4. THE README.md SHALL document all environment variables required by the Frontend, Indexer, and deployment scripts; each variable SHALL be documented with its name, a one-sentence description, and an example value.
5. WHEN a developer follows the README setup steps on a clean machine with Node.js 18+ and Git installed, THE smart contracts SHALL compile without errors, THE Indexer SHALL start without errors, and all tests SHALL pass without additional manual configuration beyond creating `.env` files from the provided `.env.example` templates.
