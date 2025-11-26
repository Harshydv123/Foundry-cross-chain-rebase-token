# 🦁 Cross-Chain Rebase Token (CCIP + Vault + TokenPool)

A cross-chain **rebasing ERC20** token that:

- accrues **per-user interest over time**,
- lets users **deposit / redeem ETH** via a Vault,
- and bridges across chains using **Chainlink CCIP Token Pools**.

Built with **Solidity 0.8.24**, **Foundry**, **OpenZeppelin**, and **Chainlink CCIP**.

---

## ✨ High-Level Overview

This project implements a **yield-bearing, cross-chain token** with:

1. **RebaseToken**  
   - An ERC20 that **rebases linearly over time** using a per-user interest rate.
   - Interest rate for *new deposits* can only **decrease**, never increase.
   - Each user’s interest rate is locked in when they first receive tokens.

2. **Vault**  
   - Accepts **native ETH deposits**.
   - Mints RebaseToken to the depositor at the current interest rate.
   - Allows **redemption** of tokens back into ETH.
   - Can receive additional ETH rewards to fund accrued interest.

3. **RebaseTokenPool (CCIP Token Pool)**  
   - Integrates with **Chainlink CCIP** to bridge RebaseToken across chains.
   - On the source chain: **burns** tokens and passes the user’s interest rate.
   - On the destination chain: **mints** tokens with the **same user interest rate**, preserving yield behavior cross-chain.

---

## 🧱 Architecture

### Components

- **RebaseToken.sol**
  - Inherits: `ERC20`, `Ownable`, `AccessControl`.
  - Tracks:
    - `s_interestRate` – global interest rate for *future* depositors.
    - `s_userInterestRate[user]` – per-user interest rate at time of last update.
    - `s_userLastUpdatedTimestamp[user]` – last time interest was minted for a user.
  - Provides:
    - `mint(address to, uint256 value, uint256 userInterestRate)` – only callable by MINT_AND_BURN_ROLE.
    - `burn(address from, uint256 value)` – only callable by MINT_AND_BURN_ROLE.
    - `balanceOf(user)` – returns **principal + accrued interest**.
    - `principalBalanceOf(user)` – returns the stored principal (no fresh interest).
    - `setInterestRate(newRate)` – global rate setter, only owner, **can only decrease**.
    - `getInterestRate()` and `getUserInterestRate(user)` getters.

- **Vault.sol**
  - Holds ETH backing for the RebaseToken.
  - Functions:
    - `deposit()` (payable):  
      - mints RebaseToken to `msg.sender` equal to `msg.value`
      - uses the current `getInterestRate()` from RebaseToken.
    - `redeem(uint256 amount)`:
      - burns RebaseTokens from the user.
      - sends equivalent ETH back to the user.
      - if `amount == type(uint256).max`, redeems **full balance**.
  - `receive()` is implemented so the Vault can receive **extra ETH rewards** (e.g. protocol yield), which back the rebasing token.

- **RebaseTokenPool.sol**
  - Inherits: `TokenPool` from the CCIP contracts.
  - Connects RebaseToken to Chainlink CCIP for bridging.
  - Functions:
    - `lockOrBurn(Pool.LockOrBurnInV1 lockOrBurnIn)`:
      - Called on the **source chain** when bridging out.
      - Validates via `_validateLockOrBurn`.
      - Reads the user’s current interest rate:  
        `getUserInterestRate(lockOrBurnIn.originalSender)`.
      - Burns tokens from the pool:  
        `RebaseToken.burn(address(this), lockOrBurnIn.amount)`.
      - Returns `LockOrBurnOutV1` with:
        - `destTokenAddress`: the token on the destination chain.
        - `destPoolData`: ABI-encoded `userInterestRate`.
    - `releaseOrMint(Pool.ReleaseOrMintInV1 releaseOrMintIn)`:
      - Called on the **destination chain** when tokens arrive.
      - Validates via `_validateReleaseOrMint`.
      - Decodes `userInterestRate` from `sourcePoolData`.
      - Mints tokens to the receiver:  
        `RebaseToken.mint(receiver, amount, userInterestRate)`.

---

## 🔁 Cross-Chain Flow (CCIP)

### Bridging OUT (Source Chain)

1. User holds `X` RebaseTokens on **Chain A**.
2. User initiates cross-chain transfer via CCIP router.
3. CCIP router calls `lockOrBurn` on `RebaseTokenPool` (Chain A).
4. `lockOrBurn`:
   - Validates call (`_validateLockOrBurn`).
   - Reads user’s `userInterestRate` from `RebaseToken`.
   - Burns `X` tokens from the pool.
   - Returns `LockOrBurnOutV1` with:
     - destination token address,
     - encoded `userInterestRate`.

### Bridging IN (Destination Chain)

1. CCIP delivers message to `RebaseTokenPool` on **Chain B**.
2. CCIP router calls `releaseOrMint` with:
   - `receiver`
   - `amount = X`
   - `sourcePoolData` = encoded `userInterestRate`.
3. `releaseOrMint`:
   - Validates call (`_validateReleaseOrMint`).
   - Decodes `userInterestRate`.
   - Calls `RebaseToken.mint(receiver, X, userInterestRate)`.
4. User now holds `X` RebaseTokens on Chain B with **the same yield rate** they had on Chain A.

---

## 📊 Interest & Rebasing Model

- **Linear interest model**:
  ```solidity
  linearInterest = (userInterestRate * timeDifference) + PRECISION_FACTOR;
  updatedBalance = principal * linearInterest / PRECISION_FACTOR;
🛠 Tech Stack

Language: Solidity 0.8.24

Framework: Foundry

Libraries:

OpenZeppelin ERC20, Ownable, AccessControl

Chainlink CCIP contracts (router, token pool, pool library)

Tooling:

VS Code + Solidity extensions

forge for build/test
