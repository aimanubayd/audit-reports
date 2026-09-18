---
title: Vault Guardians Audit Report
author: Aiman Ubayd
date: Aug 25, 2026
header-includes:
  - \usepackage{titling}
  - \usepackage{graphicx}
---

\begin{titlepage}
    \centering
    \begin{figure}[h]
        \centering
        \includegraphics[width=0.5\textwidth]{logo.pdf} 
    \end{figure}
    \vspace*{2cm}
    {\Huge\bfseries Protocol Audit Report\par}
    \vspace{1cm}
    {\Large Version 1.0\par}
    \vspace{2cm}
    {\Large\itshape @aimanubayd\par}
    \vfill
    {\large \today\par}
\end{titlepage}

\maketitle

<!-- Your report starts here! -->

Prepared by: Aiman Ubayd

Assisting Auditors:

- None

# Table of Contents
- [Table of Contents](#table-of-contents)
- [About Aiman Ubayd](#about-aiman-ubayd)
- [Disclaimer](#disclaimer)
- [Risk Classification](#risk-classification)
- [Audit Details](#audit-details)
  - [Scope](#scope)
- [Protocol Summary](#protocol-summary)
  - [Roles](#roles)
- [Executive Summary](#executive-summary)
  - [Issues Found](#issues-found)
  - [High](#high)
    - [\[H-1\] Lack of UniswapV2 slippage protection in `UniswapAdapter::_uniswapInvest` enables frontrunners to steal profits (`amountOutMin` hardcoded to 0 and `deadline` set to `block.timestamp` -\> MEV bots and holding validators can force the swap through at an arbitrarily bad rate)](#h-1-lack-of-uniswapv2-slippage-protection-in-uniswapadapter_uniswapinvest-enables-frontrunners-to-steal-profits-amountoutmin-hardcoded-to-0-and-deadline-set-to-blocktimestamp---mev-bots-and-holding-validators-can-force-the-swap-through-at-an-arbitrarily-bad-rate)
    - [\[H-2\] `ERC4626::totalAssets` checks the balance of vault's underlying asset even when the asset is invested, resulting in incorrect values being returned (`totalAssets` reads only the vault's idle token balance, ignoring funds deployed to Aave/Uniswap -\> every share-price calculation in the vault is wrong)](#h-2-erc4626totalassets-checks-the-balance-of-vaults-underlying-asset-even-when-the-asset-is-invested-resulting-in-incorrect-values-being-returned-totalassets-reads-only-the-vaults-idle-token-balance-ignoring-funds-deployed-to-aaveuniswap---every-share-price-calculation-in-the-vault-is-wrong)
    - [\[H-3\] Guardians can infinitely mint `VaultGuardianToken`s and take over DAO, stealing DAO fees and maliciously setting parameters (Guardians are minted vgTokens on every `becomeGuardian` call with no vesting/cooldown, and can freely `quitGuardian` to repeat -\> vgTokens can be farmed indefinitely to seize DAO control)](#h-3-guardians-can-infinitely-mint-vaultguardiantokens-and-take-over-dao-stealing-dao-fees-and-maliciously-setting-parameters-guardians-are-minted-vgtokens-on-every-becomeguardian-call-with-no-vestingcooldown-and-can-freely-quitguardian-to-repeat---vgtokens-can-be-farmed-indefinitely-to-seize-dao-control)
  - [Medium](#medium)
    - [\[M-1\] Potentially incorrect voting period and delay in governor may affect governance (`votingDelay`/`votingPeriod` return values expressed in seconds instead of the blocks the Governor framework expects -\> actual voting delay/period is far shorter or longer than intended)](#m-1-potentially-incorrect-voting-period-and-delay-in-governor-may-affect-governance-votingdelayvotingperiod-return-values-expressed-in-seconds-instead-of-the-blocks-the-governor-framework-expects---actual-voting-delayperiod-is-far-shorter-or-longer-than-intended)
  - [Low](#low)
    - [\[L-1\] Incorrect vault name and symbol (`becomeTokenGuardian` reuses `TOKEN_ONE` naming constants for the `i_tokenTwo` branch -\> vaults for the second token are deployed with the wrong name/symbol)](#l-1-incorrect-vault-name-and-symbol-becometokenguardian-reuses-token_one-naming-constants-for-the-i_tokentwo-branch---vaults-for-the-second-token-are-deployed-with-the-wrong-namesymbol)
    - [\[L-2\] Unassigned return value when divesting AAVE funds (`_aaveDivest` never assigns to its named return variable -\> `amountOfAssetReturned` always reports zero regardless of the actual amount withdrawn)](#l-2-unassigned-return-value-when-divesting-aave-funds-_aavedivest-never-assigns-to-its-named-return-variable---amountofassetreturned-always-reports-zero-regardless-of-the-actual-amount-withdrawn)


# About Aiman Ubayd

Aiman Ubayd is a smart contract auditor and security researcher. He researches how EVM & DeFi protocols fail, tracing exploit paths in protocol logic, economics, and trust boundaries. He combines manual security research with fuzzing, invariant testing, and formal verification, backed by AI-powered tooling and AI agents for web3 security analysis.

- **X (Twitter):** [https://x.com/aimanubayd]
- **GitHub:** [https://github.com/aimanubayd]
- **LinkedIn:** [https://linkedin.com/in/aimanubayd]
- **Contact:** [aimanubayd@gmail.com]

# Disclaimer

Aiman Ubayd makes all effort to find as many vulnerabilities in the code in the given time period, but holds no responsibilities for the findings provided in this document. A security audit by the team is not an endorsement of the underlying business or product. The audit was time-boxed and the review of the code was solely on the security aspects of the Solidity implementation of the contracts.

# Risk Classification

|            |        | Impact |        |     |
| ---------- | ------ | ------ | ------ | --- |
|            |        | High   | Medium | Low |
|            | High   | H      | H/M    | M   |
| Likelihood | Medium | H/M    | M      | M/L |
|            | Low    | M      | M/L    | L   |


# Audit Details 

**The findings described in this document correspond the following commit hash:**
```
daa885d5cb05b4fe1c48f9708ac1c096b372f3c0
```

## Scope 

```
./src/
#-- abstract
|   #-- AStaticTokenData.sol
|   #-- AStaticUSDCData.sol
|   #-- AStaticWethData.sol
#-- dao
|   #-- VaultGuardianGovernor.sol
|   #-- VaultGuardianToken.sol
#-- interfaces
|   #-- IVaultData.sol
|   #-- IVaultGuardians.sol
|   #-- IVaultShares.sol
|   #-- InvestableUniverseAdapter.sol
#-- protocol
|   #-- VaultGuardians.sol
|   #-- VaultGuardiansBase.sol
|   #-- VaultShares.sol
|   #-- investableUniverseAdapters
|       #-- AaveAdapter.sol
|       #-- UniswapAdapter.sol
#-- vendor
    #-- DataTypes.sol
    #-- IPool.sol
    #-- IUniswapV2Factory.sol
    #-- IUniswapV2Router01.sol
```

# Protocol Summary 

This protocol allows users to deposit certain ERC20s into an [ERC4626 vault](https://eips.ethereum.org/EIPS/eip-4626) managed by a human being, or a `vaultGuardian`. The goal of a `vaultGuardian` is to manage the vault in a way that maximizes the value of the vault for the users who have despoited money into the vault.

## Roles

There are 4 main roles associated with the system. 

- *Vault Guardian DAO*: The org that takes a cut of all profits, controlled by the `VaultGuardianToken`. The DAO that controls a few variables of the protocol, including:
  - `s_guardianStakePrice`
  - `s_guardianAndDaoCut`
  - And takes a cut of the ERC20s made from the protocol
- *DAO Participants*: Holders of the `VaultGuardianToken` who vote and take profits on the protocol
- *Vault Guardians*: Strategists/hedge fund managers who have the ability to move assets in and out of the investable universe. They take a cut of revenue from the protocol. 
- *Investors*: The users of the protocol. They deposit assets to gain yield from the investments of the Vault Guardians. 

# Executive Summary

The Vault Guardians project takes novel approaches to work ERC-4626 into a hedge fund of sorts, but makes some large mistakes on tracking balances and profits. 

## Issues Found

| Severity | Number of issues found |
| -------- | ---------------------- |
| High     | 3                      |
| Medium   | 1                      |
| Low      | 2                      |
| Info     | 0                      |
| Gas      | 0                      |
| Total    | 6                      |

## High

### [H-1] Lack of UniswapV2 slippage protection in `UniswapAdapter::_uniswapInvest` enables frontrunners to steal profits (`amountOutMin` hardcoded to 0 and `deadline` set to `block.timestamp` -> MEV bots and holding validators can force the swap through at an arbitrarily bad rate)

**Description:** In `UniswapAdapter::_uniswapInvest` the protocol swaps half of an ERC20 token so that they can invest in both sides of a Uniswap pool. It calls the `swapExactTokensForTokens` function of the `UnisapV2Router01` contract , which has two input parameters to note:

```javascript
    function swapExactTokensForTokens(
        uint256 amountIn,
@>      uint256 amountOutMin,
        address[] calldata path,
        address to,
@>      uint256 deadline
    )
```

The parameter `amountOutMin` represents how much of the minimum number of tokens it expects to return. 
The `deadline` parameter represents when the transaction should expire.

As seen below, the `UniswapAdapter::_uniswapInvest` function sets those parameters to `0` and `block.timestamp`:

```javascript
    uint256[] memory amounts = i_uniswapRouter.swapExactTokensForTokens(
        amountOfTokenToSwap, 
@>      0, 
        s_pathArray, 
        address(this), 
@>      block.timestamp
    );
```

**Impact:** This results in either of the following happening:
- Anyone (e.g., a frontrunning bot) sees this transaction in the mempool, pulls a flashloan and swaps on Uniswap to tank the price before the swap happens, resulting in the protocol executing the swap at an unfavorable rate.
- Due to the lack of a deadline, the node who gets this transaction could hold the transaction until they are able to profit from the guaranteed swap.

**Proof of Concept:**

1. User calls `VaultShares::deposit` with a vault that has a Uniswap allocation. 
   1. This calls `_uniswapInvest` for a user to invest into Uniswap, and calls the router's `swapExactTokensForTokens` function.
2. In the mempool, a malicious user could:
   1. Hold onto this transaction which makes the Uniswap swap
   2. Take a flashloan out
   3. Make a major swap on Uniswap, greatly changing the price of the assets
   4. Execute the transaction that was being held, giving the protocol as little funds back as possible due to the `amountOutMin` value set to 0. 

This could potentially allow malicious MEV users and frontrunners to drain balances. 

**Recommended Mitigation:** 

*For the deadline issue, we recommend the following:*

DeFi is a large landscape. For protocols that have sensitive investing parameters, add a custom parameter to the `deposit` function so the Vault Guardians protocol can account for the customizations of DeFi projects that it integrates with.

In the `deposit` function, consider allowing for custom data. 

```diff
- function deposit(uint256 assets, address receiver) public override(ERC4626, IERC4626) isActive returns (uint256) {
+ function deposit(uint256 assets, address receiver, bytes customData) public override(ERC4626, IERC4626) isActive returns (uint256) {  
```

This way, you could add a `deadline` to the Uniswap swap, and also allow for more DeFi custom integrations. 

*For the `amountOutMin` issue, we recommend one of the following:*

1. Do a price check on something like a [Chainlink price feed](https://docs.chain.link/data-feeds) before making the swap, reverting if the rate is too unfavorable.
2. Only deposit 1 side of a Uniswap pool for liquidity. Don't make the swap at all. If a pool doesn't exist or has too low liquidity for a pair of ERC20s, don't allow investment in that pool. 

Note that these recommendation require significant changes to the codebase.

### [H-2] `ERC4626::totalAssets` checks the balance of vault's underlying asset even when the asset is invested, resulting in incorrect values being returned (`totalAssets` reads only the vault's idle token balance, ignoring funds deployed to Aave/Uniswap -> every share-price calculation in the vault is wrong)

**Description:** The `ERC4626::totalAssets` function checks the balance of the underlying asset for the vault using the `balanceOf` function. 

```javascript
function totalAssets() public view virtual returns (uint256) {
    return _asset.balanceOf(address(this));
}
```

However, the assets are invested in the investable universe (Aave and Uniswap) which means this will never return the correct value of assets in the vault. 

**Impact:** This breaks many functions of the `ERC4626` contract:
- `totalAssets`
- `convertToShares`
- `convertToAssets`
- `previewWithdraw`
- `withdraw`
- `deposit`

All calculations that depend on the number of assets in the protocol would be flawed, severely disrupting the protocol functionality.  

**Proof of Concept:**

<details>
<summary>Code</summary>

Add the following code to the `VaultSharesTest.t.sol` file. 

```javascript
function testWrongBalance() public {
    // Mint 100 ETH
    weth.mint(mintAmount, guardian);
    vm.startPrank(guardian);
    weth.approve(address(vaultGuardians), mintAmount);
    address wethVault = vaultGuardians.becomeGuardian(allocationData);
    wethVaultShares = VaultShares(wethVault);
    vm.stopPrank();

    // prints 3.75 ETH
    console.log(wethVaultShares.totalAssets());

    // Mint another 100 ETH
    weth.mint(mintAmount, user);
    vm.startPrank(user);
    weth.approve(address(wethVaultShares), mintAmount);
    wethVaultShares.deposit(mintAmount, user);
    vm.stopPrank();

    // prints 41.25 ETH
    console.log(wethVaultShares.totalAssets());
}
```
</details>

**Recommended Mitigation:** Do not use the OpenZeppelin implementation of the `ERC4626` contract. Instead, natively keep track of users total amounts sent to each protocol. Potentially have an automation tool or some incentivised mechanism to keep track of protocol's profits and losses, and take snapshots of the investable universe.

This would take a considerable re-write of the protocol. 

### [H-3] Guardians can infinitely mint `VaultGuardianToken`s and take over DAO, stealing DAO fees and maliciously setting parameters (Guardians are minted vgTokens on every `becomeGuardian` call with no vesting/cooldown, and can freely `quitGuardian` to repeat -> vgTokens can be farmed indefinitely to seize DAO control)

**Description:** Becoming a guardian comes with the perk of getting minted Vault Guardian Tokens (vgTokens). Whenever a guardian successfully calls `VaultGuardiansBase::becomeGuardian` or `VaultGuardiansBase::becomeTokenGuardian`, `_becomeTokenGuardian` is executed, which mints the caller `i_vgToken`. 

```javascript
    function _becomeTokenGuardian(IERC20 token, VaultShares tokenVault) private returns (address) {
        s_guardians[msg.sender][token] = IVaultShares(address(tokenVault));
@>      i_vgToken.mint(msg.sender, s_guardianStakePrice);
        emit GuardianAdded(msg.sender, token);
        token.safeTransferFrom(msg.sender, address(this), s_guardianStakePrice);
        token.approve(address(tokenVault), s_guardianStakePrice);
        tokenVault.deposit(s_guardianStakePrice, msg.sender);
        return address(tokenVault);
    }
```

Guardians are also free to quit their role at any time, calling the `VaultGuardianBase::quitGuardian` function. The combination of minting vgTokens, and freely being able to quit, results in users being able to farm vgTokens at any time.

**Impact:** Assuming the token has no monetary value, the malicious guardian could accumulate tokens until they can overtake the DAO. Then, they could execute any of these functions of the `VaultGuardians` contract:

```
  "sweepErc20s(address)": "942d0ff9",
  "transferOwnership(address)": "f2fde38b",
  "updateGuardianAndDaoCut(uint256)": "9e8f72a4",
  "updateGuardianStakePrice(uint256)": "d16fe105",
```

**Proof of Concept:**

1. User becomes WETH guardian and is minted vgTokens.
2. User quits, is given back original WETH allocation.
3. User becomes WETH guardian with the same initial allocation.
4. Repeat to keep minting vgTokens indefinitely.

<details>
<summary>Code</summary>

Place the following code into `VaultGuardiansBaseTest.t.sol`

```javascript
    function testDaoTakeover() public hasGuardian hasTokenGuardian {
        address maliciousGuardian = makeAddr("maliciousGuardian");
        uint256 startingVoterUsdcBalance = usdc.balanceOf(maliciousGuardian);
        uint256 startingVoterWethBalance = weth.balanceOf(maliciousGuardian);
        assertEq(startingVoterUsdcBalance, 0);
        assertEq(startingVoterWethBalance, 0);

        VaultGuardianGovernor governor = VaultGuardianGovernor(payable(vaultGuardians.owner()));
        VaultGuardianToken vgToken = VaultGuardianToken(address(governor.token()));

        // Flash loan the tokens, or just buy a bunch for 1 block
        weth.mint(mintAmount, maliciousGuardian); // The same amount as the other guardians
        uint256 startingMaliciousVGTokenBalance = vgToken.balanceOf(maliciousGuardian);
        uint256 startingRegularVGTokenBalance = vgToken.balanceOf(guardian);
        console.log("Malicious vgToken Balance:\t", startingMaliciousVGTokenBalance);
        console.log("Regular vgToken Balance:\t", startingRegularVGTokenBalance);

        // Malicious Guardian farms tokens
        vm.startPrank(maliciousGuardian);
        weth.approve(address(vaultGuardians), type(uint256).max);
        for (uint256 i; i < 10; i++) {
            address maliciousWethSharesVault = vaultGuardians.becomeGuardian(allocationData);
            IERC20(maliciousWethSharesVault).approve(
                address(vaultGuardians),
                IERC20(maliciousWethSharesVault).balanceOf(maliciousGuardian)
            );
            vaultGuardians.quitGuardian();
        }
        vm.stopPrank();

        uint256 endingMaliciousVGTokenBalance = vgToken.balanceOf(maliciousGuardian);
        uint256 endingRegularVGTokenBalance = vgToken.balanceOf(guardian);
        console.log("Malicious vgToken Balance:\t", endingMaliciousVGTokenBalance);
        console.log("Regular vgToken Balance:\t", endingRegularVGTokenBalance);
    }
```
</details>

**Recommended Mitigation:** There are a few options to fix this issue:

1. Mint vgTokens on a vesting schedule after a user becomes a guardian.
2. Burn vgTokens when a guardian quits.
3. Simply don't allocate vgTokens to guardians. Instead, mint the total supply on contract deployment.

## Medium

### [M-1] Potentially incorrect voting period and delay in governor may affect governance (`votingDelay`/`votingPeriod` return values expressed in seconds instead of the blocks the Governor framework expects -> actual voting delay/period is far shorter or longer than intended)

**Description:** The `VaultGuardianGovernor` contract, based on [OpenZeppelin Contract's Governor](https://docs.openzeppelin.com/contracts/5.x/api/governance#governor), implements two functions to define the voting delay (`votingDelay`) and period (`votingPeriod`). The contract intends to define a voting delay of 1 day, and a voting period of 7 days. It does it by returning the value `1 days` from `votingDelay` and `7 days` from `votingPeriod`. In Solidity these values are translated to number of seconds.

However, the `votingPeriod` and `votingDelay` functions, by default, are expected to return number of blocks, not the number of seconds.

**Impact:** Since `1 days` (86,400) and `7 days` (604,800) are being interpreted as block counts rather than seconds, the actual voting delay and period will be far off from what the developers intended — orders of magnitude longer than the intended 1 day / 7 days, given typical block times. This could potentially affect the intended governance mechanics, for example by making proposals take an impractically long time to become votable or to resolve, or otherwise deviating from the governance cadence the DAO participants expect.

**Proof of Concept:** With an approximate 12-second block time, a `votingDelay()` of `1 days` (86,400) resolves to roughly 86,400 blocks × 12 seconds ~ 12 days before voting opens, instead of the intended 1 day. Similarly, `votingPeriod()` of `7 days` (604,800 blocks) resolves to roughly 604,800 × 12 seconds ~ 84 days of voting, instead of the intended 7 days.

**Recommended Mitigation:** Consider updating the functions as follows:

```diff
function votingDelay() public pure override returns (uint256) {
-   return 1 days;
+   return 7200; // 1 day
}

function votingPeriod() public pure override returns (uint256) {
-   return 7 days;
+   return 50400; // 1 week
}
```

## Low

### [L-1] Incorrect vault name and symbol (`becomeTokenGuardian` reuses `TOKEN_ONE` naming constants for the `i_tokenTwo` branch -> vaults for the second token are deployed with the wrong name/symbol)

**Description:** When new vaults are deployed in the `VaultGuardianBase::becomeTokenGuardian` function, symbol and vault name are set incorrectly when the `token` is equal to `i_tokenTwo` — the branch handling `i_tokenTwo` still references the `TOKEN_ONE_VAULT_NAME`/`TOKEN_ONE_VAULT_SYMBOL` constants instead of the `TOKEN_TWO` equivalents.

**Impact:** Vaults deployed for `i_tokenTwo` will be misidentified by their own name and symbol, which can cause errors in off-chain clients, block explorers, and wallets that read these values to identify and display vaults, potentially leading users to confuse the `i_tokenTwo` vault for the `i_tokenOne` vault.

**Proof of Concept:** Call `becomeTokenGuardian` with `token == i_tokenTwo` and inspect the resulting `VaultShares` contract's `name()`/`symbol()` — they will read as the `TOKEN_ONE` constants rather than the `TOKEN_TWO` constants.

**Recommended Mitigation:** Consider modifying the function as follows, to avoid errors in off-chain clients reading these values to identify vaults. 

```diff
else if (address(token) == address(i_tokenTwo)) {
    tokenVault =
    new VaultShares(IVaultShares.ConstructorData({
        asset: token,
-       vaultName: TOKEN_ONE_VAULT_NAME,
+       vaultName: TOKEN_TWO_VAULT_NAME,
-       vaultSymbol: TOKEN_ONE_VAULT_SYMBOL,
+       vaultSymbol: TOKEN_TWO_VAULT_SYMBOL,
        guardian: msg.sender,
        allocationData: allocationData,
        aavePool: i_aavePool,
        uniswapRouter: i_uniswapV2Router,
        guardianAndDaoCut: s_guardianAndDaoCut,
        vaultGuardian: address(this),
        weth: address(i_weth),
        usdc: address(i_tokenOne)
    }));
```

Also, add a new test in the `VaultGuardiansBaseTest.t.sol` file to avoid reintroducing this error, similar to what's done in the test `testBecomeTokenGuardianTokenOneName`.

### [L-2] Unassigned return value when divesting AAVE funds (`_aaveDivest` never assigns to its named return variable -> `amountOfAssetReturned` always reports zero regardless of the actual amount withdrawn)

**Description:** The `AaveAdapter::_aaveDivest` function is intended to return the amount of assets returned by AAVE after calling its `withdraw` function. However, the code never assigns a value to the named return variable `amountOfAssetReturned`.

**Impact:** As a result, `_aaveDivest` will always return zero, regardless of how much was actually withdrawn from Aave. While this return value is not currently being used anywhere in the code, any future change that relies on this function's return value to track divested amounts would silently receive incorrect data.

**Proof of Concept:** Call `_aaveDivest` with a nonzero `amount` for a token actually deposited in Aave, and observe that the function's return value is `0` despite Aave's `withdraw` call actually returning and transferring a nonzero amount of the underlying asset.

**Recommended Mitigation:** Update the `_aaveDivest` function as follows:

```diff
function _aaveDivest(IERC20 token, uint256 amount) internal returns (uint256 amountOfAssetReturned) {
-       i_aavePool.withdraw({
+       amountOfAssetReturned = i_aavePool.withdraw({
            asset: address(token),
            amount: amount,
            to: address(this)
        });
}
```