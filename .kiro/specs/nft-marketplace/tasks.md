# Implementation Plan: NFT Marketplace with Fractionalization

## Overview

Монорепозиторій реалізується поетапно: спочатку налаштовується проєктна структура та інфраструктура, потім смарт-контракти (NFTCollection → Marketplace → FractionVault/FractionToken → NFTGovernor → Paymaster), далі — Indexer, Frontend, CI/CD та документація. Кожен крок будується на попередньому і завершується інтеграцією всіх шарів.

---

## Tasks

- [ ] 1. Налаштування монорепозиторію та базової інфраструктури
  - Ініціалізувати кореневий `package.json` з workspaces (`contracts`, `indexer`, `frontend`)
  - Створити `hardhat.config.ts` з підтримкою zkSync Era Sepolia (chain ID 300, RPC, `@matterlabs/hardhat-zksync`)
  - Додати `.solhint.json` та `.eslintrc.json` з правилами для Solidity та TypeScript
  - Створити `.env.example` для кореня, `indexer/.env.example`, `frontend/.env.example` з усіма змінними середовища
  - Створити `LICENSE` (MIT)
  - _Requirements: 10.1, 11.3, 11.4, 11.5_

  - [ ] 1.1 Ініціалізувати Hardhat-проєкт зі структурою директорій
    - Створити `contracts/`, `test/`, `test/integration/`, `scripts/`, `deployments/`
    - Встановити залежності: `hardhat`, `@matterlabs/hardhat-zksync`, `@openzeppelin/contracts`, `erc721a`, `ethers`, `typescript`, `ts-node`, `@types/node`
    - Налаштувати `tsconfig.json` для Hardhat
    - _Requirements: 9.1, 10.1_

  - [ ] 1.2 Ініціалізувати Indexer-проєкт
    - Створити `indexer/src/` зі структурою: `index.ts`, `listener.ts`, `api/router.ts`, `db/schema.sql`, `db/queries.ts`, `validators/eventSchemas.ts`
    - Встановити залежності: `express`, `ethers`, `pg`, `zod`, `cors`, `dotenv`, `typescript`
    - Налаштувати `indexer/tsconfig.json`
    - _Requirements: 8.1, 8.2_

  - [ ] 1.3 Ініціалізувати Frontend-проєкт
    - Створити Next.js 14 App Router проєкт у `frontend/`
    - Встановити залежності: `wagmi`, `viem`, `@rainbow-me/rainbowkit`, `@tanstack/react-query`, `tailwindcss`, `web3.storage`
    - Налаштувати `frontend/lib/wagmi.ts` з конфігурацією zkSync Era Sepolia та трьома конекторами (injected, walletConnect, coinbaseWallet)
    - _Requirements: 7.1, 7.2_

- [ ] 2. Смарт-контракт NFTCollection
  - [ ] 2.1 Реалізувати `contracts/NFTCollection.sol`
    - Успадкувати `ERC721A`, `ERC2981`
    - Реалізувати `mint(address to, uint256 quantity, string[] calldata uris, uint96 royaltyBps)`:
      - Валідація: `quantity` 1–20, кожен URI непорожній та починається з `ipfs://`, `royaltyBps` ≤ 10000
      - Мінтинг через `_mint`, збереження URI у `_tokenURIs[tokenId]`, виклик `_setTokenRoyalty` для кожного токена
    - Реалізувати `tokenURI(uint256 tokenId)` з перевіркою існування токена
    - Перевизначити `supportsInterface` для `ERC721A` + `ERC2981`
    - _Requirements: 1.1, 1.2, 1.3, 1.4, 1.5, 1.6, 1.7, 1.8, 1.9, 1.10_

  - [ ]* 2.2 Написати unit-тести для NFTCollection (`test/NFTCollection.test.ts`)
    - Тести успішного мінтингу (одиночний та batch 1–20)
    - Тести revert: порожній URI, URI без `ipfs://`, `royaltyBps > 10000`, `quantity = 0`, `quantity > 20`, `tokenURI` для неіснуючого tokenId
    - Тест `royaltyInfo`: перевірити повернення правильного recipient та суми
    - Тест `tokenURI`: повертає точний URI, переданий при мінтингу (Requirements 9.6)
    - Досягти ≥ 90% line/branch coverage для контракту
    - _Requirements: 9.1, 9.2, 9.6_

- [ ] 3. Смарт-контракт Marketplace
  - [ ] 3.1 Реалізувати `contracts/Marketplace.sol`
    - Успадкувати `ReentrancyGuard`, `Pausable`, `Ownable`
    - Структура `Listing { address seller; address tokenAddress; uint256 tokenId; uint256 price; }`
    - Маппінг `listings: tokenAddress → tokenId → Listing`
    - Реалізувати `listNFT(address tokenAddress, uint256 tokenId, uint256 price)`:
      - Перевірити ownership, approval, `price > 0`, відсутність активного лістингу
      - Emit `Listed(tokenId, seller, price)`
    - Реалізувати `cancelListing(address tokenAddress, uint256 tokenId)`:
      - Перевірити, що caller == listing.seller
      - Emit `ListingCancelled`
    - Реалізувати `buyNFT(address tokenAddress, uint256 tokenId) payable nonReentrant`:
      - CEI: завантажити лістинг → видалити → перевірити `msg.value == price` → `royaltyInfo` → revert якщо `royaltyAmount >= price` → `safeTransferFrom` → transfer royalty → transfer remainder → emit `Sold`
    - _Requirements: 3.1, 3.2, 3.3, 3.4, 3.5, 3.6, 3.7, 3.8, 3.9, 3.10, 3.11, 3.12, 3.13_

  - [ ]* 3.2 Написати unit-тести для Marketplace (`test/Marketplace.test.ts`)
    - Тести `listNFT`: успішний лістинг, revert без approval, revert без ownership, revert при `price = 0`, revert при дублікаті лістингу
    - Тести `buyNFT`: успішна купівля з перевіркою балансів seller/royalty, revert при неправильній сумі ETH, revert для неіснуючого лістингу, revert якщо `royaltyAmount >= price`
    - Тести `cancelListing`: успішне скасування, revert для не-seller
    - Тест: querying listing повертає правильний price та seller (Requirements 9.7)
    - Досягти ≥ 90% line/branch coverage
    - _Requirements: 9.1, 9.2, 9.7_

- [ ] 4. Смарт-контракти FractionVault та FractionToken
  - [ ] 4.1 Реалізувати `contracts/FractionToken.sol`
    - Успадкувати `ERC20`, `ERC20Votes`
    - Конструктор: `(string name, string symbol, address recipient, uint256 supply, address _vault)`
    - Зберігати `address public immutable vault`
    - Мінтувати `supply` токенів на `recipient` у конструкторі
    - Реалізувати `burnAll(address from) external` — тільки `vault` може викликати, спалює весь supply
    - _Requirements: 4.7, 5.8_

  - [ ] 4.2 Реалізувати `contracts/FractionVault.sol`
    - Успадкувати `ReentrancyGuard`, `IERC721Receiver`
    - Структура `VaultEntry { address nftAddress; uint256 tokenId; address depositor; address fractionToken; bool active; }`
    - Маппінг `vaults: nftAddress → tokenId → VaultEntry`
    - Реалізувати `deposit(address nftAddress, uint256 tokenId)`:
      - Перевірити `!vaults[nftAddress][tokenId].active`
      - `safeTransferFrom` NFT до vault
      - Задеплоїти новий `FractionToken(name, symbol, msg.sender, 1_000_000 * 1e18, address(this))`
      - Зберегти `VaultEntry`, emit `Deposited`
    - Реалізувати `redeem(address nftAddress, uint256 tokenId) nonReentrant`:
      - Перевірити `active == true`
      - Перевірити `fractionToken.balanceOf(msg.sender) == fractionToken.totalSupply()`
      - `fractionToken.burnAll(msg.sender)`
      - `vault.active = false`
      - `safeTransferFrom` NFT назад до caller
      - Emit `Redeemed`
    - Реалізувати `onERC721Received` (повертає selector)
    - _Requirements: 4.1, 4.2, 4.3, 4.4, 4.5, 4.6, 4.8, 4.9_

  - [ ]* 4.3 Написати unit-тести для FractionVault та FractionToken (`test/FractionVault.test.ts`)
    - Тести `deposit`: успішний депозит, revert без approval, revert при повторному депозиті того ж tokenId
    - Тести `redeem`: успішний redeem при 100% supply, revert при < 100% supply
    - Тест: після депозиту сума балансів усіх holders == 1,000,000 × 10^18 (Requirements 9.8)
    - Тест: після redeem vault.active == false, NFT повернуто
    - Тест: FractionToken.burnAll revert якщо caller != vault
    - Досягти ≥ 90% line/branch coverage
    - _Requirements: 9.1, 9.2, 9.8_

- [ ] 5. Смарт-контракт NFTGovernor
  - [ ] 5.1 Реалізувати `contracts/NFTGovernor.sol`
    - Успадкувати `Governor`, `GovernorSettings`, `GovernorCountingSimple`, `GovernorVotes`, `GovernorVotesQuorumFraction`
    - Конструктор: `(IVotes _token, uint256 _quorumNumerator)` з `GovernorSettings(1, 50400, 0)`
    - `quorumNumerator` = 1 (1%), configurable
    - Перевизначити `propose` для перевірки 1% threshold від totalSupply на snapshot block
    - Без Timelock для MVP
    - _Requirements: 5.1, 5.2, 5.3, 5.4, 5.5, 5.6, 5.7, 5.8, 5.9_

  - [ ]* 5.2 Написати unit-тести для NFTGovernor (`test/NFTGovernor.test.ts`)
    - Тест `propose`: успішне створення, revert якщо voting power < 1%
    - Тест `castVote`: запис голосу, revert при повторному голосуванні, revert після закінчення voting period
    - Тест: proposal → `Succeeded` при досягненні quorum та majority
    - Тест `execute`: успішне виконання Succeeded proposal, revert для не-Succeeded
    - Досягти ≥ 90% line/branch coverage
    - _Requirements: 9.1, 9.2_

- [ ] 6. Checkpoint — Смарт-контракти
  - Переконатися, що всі контракти компілюються без помилок (`npx hardhat compile`)
  - Переконатися, що всі unit-тести проходять (`npx hardhat test`)
  - Перевірити coverage ≥ 90% line/branch для кожного контракту (`npx hardhat coverage`)
  - Запитати користувача, якщо виникнуть питання.

- [ ] 7. Інтеграційні тести
  - [ ] 7.1 Написати інтеграційний тест mint → list → buy (`test/integration/buyFlow.test.ts`)
    - Задеплоїти NFTCollection та Marketplace
    - Mint NFT → approve Marketplace → listNFT → buyNFT
    - Перевірити: NFT ownership перейшов до Buyer, Seller отримав `price − royalty` ETH, Royalty_Recipient отримав `royalty` ETH
    - _Requirements: 9.3_

  - [ ] 7.2 Написати інтеграційний тест mint → deposit → redeem (`test/integration/fractionalizeFlow.test.ts`)
    - Задеплоїти NFTCollection та FractionVault
    - Mint NFT → approve Vault → deposit → перевірити NFT locked та Fraction_Token supply == 1,000,000 × 10^18
    - Redeem → перевірити NFT повернуто, vault.active == false
    - _Requirements: 9.4_

  - [ ] 7.3 Написати інтеграційний тест mint → deposit → DAO propose → vote → execute (`test/integration/daoFlow.test.ts`)
    - Задеплоїти NFTCollection, FractionVault, NFTGovernor
    - Mint → deposit → self-delegate → propose → castVote → mine blocks → execute
    - Перевірити: proposal створено з правильними параметрами, голоси записані на snapshot block, proposal → `Succeeded`
    - _Requirements: 9.5_

- [ ] 8. Опціональний Paymaster (`contracts/paymaster/NFTPaymaster.sol`)
  - [ ] 8.1 Реалізувати `NFTPaymaster.sol`
    - Успадкувати `BasePaymaster` (з `@account-abstraction/contracts` або zkSync-native `IPaymaster`)
    - Маппінг `whitelistedSelectors: bytes4 → bool`
    - Реалізувати `validatePaymasterUserOp`: перевірити selector у whitelist, перевірити достатність депозиту
    - Whitelist selectors: `mint`, `listNFT`, `buyNFT`, `deposit`, `redeem`, `castVote`
    - Логіка відхилення при депозиті нижче порогу (10 × estimatedGasPerOp × baseFee)
    - _Requirements: 6.1, 6.2, 6.3, 6.4, 6.5_

- [ ] 9. Deploy-скрипт та конфігурація деплою
  - [ ] 9.1 Реалізувати `scripts/deploy.ts`
    - Порядок деплою: NFTCollection → Marketplace → FractionVault → demo FractionToken → NFTGovernor(demoToken, 1) → [NFTPaymaster]
    - Записати адреси у `deployments/zkSyncEraSepolia.json`
    - _Requirements: 10.2, 10.3_

  - [ ] 9.2 Реалізувати `scripts/verify.ts`
    - Верифікація контрактів на zkSync Era Sepolia Explorer через Hardhat verify plugin
    - _Requirements: 10.2_

- [ ] 10. Indexer — база даних та listener
  - [ ] 10.1 Реалізувати схему БД та queries (`indexer/src/db/schema.sql`, `indexer/src/db/queries.ts`)
    - Створити таблиці: `indexed_state`, `listings`, `fractions`, `proposals`, `events` згідно з дизайном
    - Реалізувати функції queries: upsert listing, update listing status, upsert fraction, upsert proposal, insert event, get/set lastBlock
    - _Requirements: 8.1, 8.4_

  - [ ] 10.2 Реалізувати zod-схеми валідації подій (`indexer/src/validators/eventSchemas.ts`)
    - Схеми для: `Listed`, `Sold`, `ListingCancelled`, `Deposited`, `Redeemed`, `ProposalCreated`, `VoteCast`, `ProposalExecuted`
    - _Requirements: 8.5_

  - [ ] 10.3 Реалізувати event listener (`indexer/src/listener.ts`)
    - Підписатися на події всіх контрактів через ethers.js v6
    - Для кожної події: валідувати через zod → upsert у відповідну таблицю → оновити `lastBlock` → insert у `events`
    - Логіка reconnect: exponential backoff (1s, 2s, 4s, 8s, 16s, max 5 retries), логування кожної спроби
    - При старті: читати `lastBlock` з БД, якщо є — resume з нього; якщо ні — стартувати з `DEPLOYMENT_BLOCK`
    - _Requirements: 8.1, 8.3, 8.4, 8.5_

- [ ] 11. Indexer — REST API
  - [ ] 11.1 Реалізувати Express router (`indexer/src/api/router.ts`)
    - `GET /listings?limit=&offset=` — активні лістинги, пагінація (default limit=20, max=100)
    - `GET /listings/:tokenAddress/:tokenId` — один лістинг
    - `GET /fractions?limit=&offset=` — активні vault-и
    - `GET /fractions/:tokenAddress/:tokenId` — один vault
    - `GET /proposals?limit=&offset=` — всі proposals
    - `GET /proposals/:proposalId` — один proposal з vote counts
    - `GET /events?limit=&offset=&event=` — paginated event log
    - CORS headers для `FRONTEND_ORIGIN`
    - _Requirements: 8.2, 8.6_

  - [ ] 11.2 Реалізувати точку входу Indexer (`indexer/src/index.ts`)
    - Підключення до PostgreSQL, запуск listener, запуск Express сервера на `PORT`
    - _Requirements: 8.1, 8.2_

- [ ] 12. Checkpoint — Indexer
  - Переконатися, що Indexer компілюється без помилок (`tsc --noEmit`)
  - Переконатися, що всі REST endpoints відповідають очікуваній схемі
  - Запитати користувача, якщо виникнуть питання.

- [ ] 13. Frontend — базові компоненти та хуки
  - [ ] 13.1 Реалізувати Wagmi конфіг та провайдери (`frontend/lib/wagmi.ts`, `frontend/app/layout.tsx`)
    - Налаштувати `createConfig` з zkSync Era Sepolia, трьома конекторами
    - Обгорнути додаток у `WagmiProvider`, `RainbowKitProvider`, `QueryClientProvider`
    - _Requirements: 7.1_

  - [ ] 13.2 Реалізувати IPFS upload утиліту (`frontend/lib/ipfs.ts`)
    - `uploadToIPFS(image: File, metadata: NFTMetadata): Promise<string>`
    - Крок 1: upload image → image CID; Крок 2: build metadata JSON; Крок 3: upload metadata → metadata CID; Крок 4: return `ipfs://{metadataCID}`
    - Валідація: перевірити наявність `name`, `description`, `image` перед upload
    - _Requirements: 2.1, 2.2, 2.3, 2.4, 2.5, 2.6_

  - [ ] 13.3 Реалізувати хуки для контрактів (`frontend/hooks/`)
    - `useNFTCollection.ts`: `useMint()`
    - `useMarketplace.ts`: `useListNFT()`, `useBuyNFT()`, `useCancelListing()`
    - `useFractionVault.ts`: `useDeposit()`, `useRedeem()`
    - `useDAO.ts`: `usePropose()`, `useCastVote()`, `useDelegate()`
    - `useContractEvents.ts`: підписка на `Listed`, `Sold`, `Deposited`, `Redeemed`, `ListingCancelled` → інвалідація TanStack Query cache
    - _Requirements: 7.4, 7.5, 7.6, 7.7, 7.8, 7.9, 7.10_

- [ ] 14. Frontend — сторінки та UI flows
  - [ ] 14.1 Реалізувати компонент підключення гаманця та network banner
    - Відображати connected address, native balance, network name
    - Banner при підключенні до непідтримуваної мережі з кнопкою dismiss
    - _Requirements: 7.2, 7.3_

  - [ ] 14.2 Реалізувати сторінку Mint (`frontend/app/mint/page.tsx`)
    - `<ImageUploader>`: upload image ≤ 50 MB до IPFS
    - `<MetadataForm>`: поля `name`, `description`, `attributes`; валідація обов'язкових полів
    - `<MintButton>`: виклик `useMint()`, відображення tx hash та нового token ID після підтвердження
    - Відображення помилки при невдалому IPFS upload або revert транзакції
    - _Requirements: 7.4, 2.4, 2.5_

  - [ ] 14.3 Реалізувати сторінку Marketplace (`frontend/app/page.tsx`)
    - `<ListingsGrid>`: читає активні лістинги з Indexer API
    - `<ListingCard>`: відображає NFT, ціну, кнопку Buy
    - Buy flow: `useBuyNFT()` з правильним `msg.value`, підтвердження після tx
    - _Requirements: 7.6_

  - [ ] 14.4 Реалізувати List flow на сторінці Marketplace або окремій сторінці
    - `<MyNFTs>`: відображає NFT у власності користувача
    - List flow: вибір NFT → введення ціни → approve Marketplace → `useListNFT()` → підтвердження
    - _Requirements: 7.5_

  - [ ] 14.5 Реалізувати сторінку Fractions (`frontend/app/fractions/page.tsx`)
    - `<FractionalizeForm>`: вибір NFT → approve Vault → `useDeposit()` → відображення Fraction_Token balance
    - `<FractionHoldings>`: відображення ERC-20 балансів
    - `<RedeemButton>`: `useRedeem()` при 100% supply
    - _Requirements: 7.7_

  - [ ] 14.6 Реалізувати сторінку DAO (`frontend/app/dao/page.tsx`)
    - `<DelegateButton>`: виклик `useDelegate()` для self-delegation
    - `<ProposalList>`: читає proposals з Indexer API
    - `<ProposalCard>`: відображає стан (`Pending`, `Active`, `Succeeded`, `Defeated`, `Executed`), кнопки голосування
    - `<CreateProposalForm>`: форма для `usePropose()` при threshold ≥ 1%
    - _Requirements: 7.8_

- [ ] 15. Checkpoint — Frontend
  - Переконатися, що Frontend компілюється без помилок (`next build`)
  - Переконатися, що всі UI flows відображають підтвердження або помилки
  - Запитати користувача, якщо виникнуть питання.

- [ ] 16. CI/CD — GitHub Actions
  - [ ] 16.1 Реалізувати `.github/workflows/ci.yml`
    - Job `lint-solidity`: `solhint 'contracts/**/*.sol'` — окремий step, блокує merge при error-level findings
    - Job `lint-typescript`: `eslint 'indexer/src/**' 'frontend/app/**' 'frontend/components/**'` — окремий step
    - Job `test`: `npx hardhat test --network hardhat`
    - Job `coverage`: `npx hardhat coverage` з репортом у PR comment
    - Тригер: кожен pull request
    - _Requirements: 10.1, 10.4, 10.5, 10.6_

  - [ ] 16.2 Реалізувати `.github/workflows/cd.yml`
    - Тригер: push до `main`, needs: [lint-solidity, lint-typescript, test]
    - Job `deploy`: `npx hardhat run scripts/deploy.ts --network zkSyncEraSepolia`
    - Після деплою: `git add deployments/zkSyncEraSepolia.json` → commit `"chore: update deployment addresses [skip ci]"` → push
    - _Requirements: 10.2, 10.3_

- [ ] 17. Документація
  - [ ] 17.1 Написати `README.md`
    - Секції: Project Overview, Architecture Diagram (ASCII або Mermaid), Prerequisites, Local Setup, Running Tests, Deploying to zkSync Era Sepolia, Frontend Usage
    - Секція "Kiro IDE": (a) відкрити папку, (b) встановити залежності, (c) запустити тести, (d) запустити локальну ноду
    - Документація всіх env-змінних (name, description, example) для Frontend, Indexer та deploy scripts
    - _Requirements: 11.1, 11.2, 11.4, 11.5_

- [ ] 18. Фінальний checkpoint
  - Переконатися, що всі тести проходять: `npx hardhat test`
  - Переконатися, що coverage ≥ 90%: `npx hardhat coverage`
  - Переконатися, що контракти компілюються: `npx hardhat compile`
  - Переконатися, що Indexer та Frontend компілюються без помилок
  - Запитати користувача, якщо виникнуть питання.

---

## Notes

- Завдання, позначені `*`, є опціональними (тести) і можуть бути пропущені для швидшого MVP
- Кожне завдання посилається на конкретні вимоги для трасування
- Checkpoint-и забезпечують інкрементальну валідацію
- Unit-тести та інтеграційні тести є взаємодоповнюючими
- Paymaster (завдання 8) є опціональним відповідно до Requirement 6
- Governor для MVP використовує один FractionToken (demo token) — відоме спрощення, задокументоване в README

## Task Dependency Graph

```json
{
  "waves": [
    { "id": 0, "tasks": ["1.1", "1.2", "1.3"] },
    { "id": 1, "tasks": ["2.1", "3.1", "4.1"] },
    { "id": 2, "tasks": ["4.2", "2.2"] },
    { "id": 3, "tasks": ["3.2", "4.3", "5.1"] },
    { "id": 4, "tasks": ["5.2", "8.1", "9.1", "9.2"] },
    { "id": 5, "tasks": ["7.1", "7.2", "7.3", "10.1"] },
    { "id": 6, "tasks": ["10.2", "10.3", "13.1", "13.2"] },
    { "id": 7, "tasks": ["11.1", "13.3"] },
    { "id": 8, "tasks": ["11.2", "14.1", "14.2", "14.3"] },
    { "id": 9, "tasks": ["14.4", "14.5", "14.6"] },
    { "id": 10, "tasks": ["16.1", "16.2", "17.1"] }
  ]
}
```
