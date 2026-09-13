---
title: PasswordStore Audit Report
author: MD AIMAN
date: August 18, 2026
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
    {\Large\itshape Independent Security Review\par}
    \vfill
    {\large \today\par}
\end{titlepage}

\maketitle

<!-- Your report starts here! -->

Prepared by: [MD AIMAN](https://github.com/mdaimanW3)
Lead Auditors:
- MD Aiman

# Table of Contents
- [Table of Contents](#table-of-contents)
- [Protocol Summary](#protocol-summary)
- [Disclaimer](#disclaimer)
- [Risk Classification](#risk-classification)
- [Audit Details](#audit-details)
  - [Scope](#scope)
  - [Roles](#roles)
- [Executive Summary](#executive-summary)
  - [Issues found](#issues-found)
- [Findings](#findings)
  - [High](#high)
    - [\[H-1\] Storing the password on-chain makes it visible to aynone, and no longer private](#h-1-storing-the-password-on-chain-makes-it-visible-to-aynone-and-no-longer-private)
  - [Likelihood \& Impact:](#likelihood--impact)
    - [\[H-2\] `PasswordStore::setPassword` has no access controll, meaning a non-owner could change the password.](#h-2-passwordstoresetpassword-has-no-access-controll-meaning-a-non-owner-could-change-the-password)
  - [Likelihood \& Impact:](#likelihood--impact-1)
  - [Informational](#informational)
    - [\[I-1\] The `PasswordStore::getPassword` natspec indicates a parameter that doesn't exist, causing the natspec to be incorrect.](#i-1-the-passwordstoregetpassword-natspec-indicates-a-parameter-that-doesnt-exist-causing-the-natspec-to-be-incorrect)
  - [Likelihood \& Impact:](#likelihood--impact-2)
  - [Gas](#gas)

# Protocol Summary

PasswordStore is a protocol dedicated to the storage and retrieval of a user's passwords. The protocol is designed to be used by a single user, and is not designed to be used by multiple users. Only the owner should be able to set and access this password.

# Disclaimer

MD Aiman makes all effort to find as many vulnerabilities in the code in the given time period, but holds no responsibility for the findings provided in this document. A security review is not an endorsement of the underlying business or product. The review was time-boxed, and the assessment of the code was solely on the security aspects of the Solidity implementation of the contracts.

# Risk Classification

|            |        | Impact |        |     |
| ---------- | ------ | ------ | ------ | --- |
|            |        | High   | Medium | Low |
|            | High   | H      | H/M    | M   |
| Likelihood | Medium | H/M    | M      | M/L |
|            | Low    | M      | M/L    | L   |

We use the [CodeHawks](https://docs.codehawks.com/hawks-auditors/how-to-evaluate-a-finding-severity) severity matrix to determine severity. See the documentation for more details.

# Audit Details 

**The findings described in this document correspond to the following commit hash:**
```
<!-- TODO: paste the actual commit hash you audited, e.g. run `git log -1 --format="%H"` in the project repo -->
```

## Scope 

```
./src/
#-- PasswordStore.sol
```

## Roles

- Owner: The user who can set the password and read the password.
- Outsiders: No one else should be able to set or read the password.

# Executive Summary

This review was conducted as an independent security assessment of the PasswordStore protocol, focusing on the single in-scope contract. The methodology combined manual code review with targeted proof-of-concept testing to validate exploitability of identified issues.

Two high-severity issues were identified, both stemming from a mismatch between the contract's intended access model and its actual implementation: private data stored on-chain is not actually private, and a state-changing function intended to be owner-only lacks any access control. One informational issue relating to incorrect NatSpec documentation was also noted.

<!-- TODO: optionally add — hours spent, number of auditors, tools used (e.g. manual review, Slither, Foundry) -->

## Issues found

| Severity | Number of issues found |
| -------- | ---------------------- |
| High     | 2                      |
| Medium   | 0                      |
| Low      | 0                      |
| Info     | 1                      |
| Total    | 3                      |


# Findings
## High

### [H-1] Storing the password on-chain makes it visible to aynone, and no longer private

**Description:** All data stored on-chain is visible to anyone, and can be read directly from the blockchain. The `PasswordStore::s_password` variable is intended to be a private variable and only accessed through the `PasswordStore::getPassword` function, which is intended to be callable only by the owner of the contract.

We show one such method of reading any data off chain below.

**Impact** Anyone can read the private password, severely breaking the functionality of the protocol.

**Proof of Concept:** (Proof of Code)

The below test case shows how anyone can read the password directly from the blockchain.

1. Create a locally running chain
```bash
make anvil
```

2. Deploy the contract to the chain

```
make deploy
```

3. Run the storage tool

We use `1` because that's the storage slot of `s_password` in the contract.

```
cast storage <ADDRESS_HERE> 1 --rpc-url http://127.0.0.1:8545
```

You'll get an output that looks like this:

`0x6d7950617373776f726400000000000000000000000000000000000000000014`

You can then parse that hex to a string with:

```
cast parse-bytes32-string 0x6d7950617373776f726400000000000000000000000000000000000000000014
```

And get an output of:

```
myPassword
```

**Recommended Mitigation:** Due to this, the overall architecture of the contract should be rethought. One could encrypt the password off-chain, and then store the encrypted password on-chain. This would require the user to remember another password off-chain to decrypt the password. However, you'd also likely want to remove the view function, as you wouldn't want the user to accidentally send a transaction with the password that decrypts their password.


## Likelihood & Impact:

- Impact: HIGH
- Likelihood: HIGH
- Severity: HIGH

### [H-2] `PasswordStore::setPassword` has no access controll, meaning a non-owner could change the password.

**Description:** The `PasswordStore::setPassword` function is set to be an `external` function, however, the NatSpec of the function and overall purpose of the smart contract is that `The function allows only the owner to set a new password.`

```javascript
function setPassword(string memory newPassword) external {
@>    // @audit - There are no access controls
        s_password = newPassword;
        emit SetNetPassword();
    }

```

**Impact:** Anyone can set/change the password of the contract, severely breaking the contract's intended functionality.

**Proof of Concept:** Add the following to the `PasswordStore.t.sol` test file.

<details>
<summary>Code</summary>

``` javascript
 function test_anyone_can_set_password(address randomAddress) public {
        vm.assume(randomAddress != owner);
        vm.prank(randomAddress);
        string memory expectedPassword = "myNewPassword";
        passwordStore.setPassword(expectedPassword);

        vm.prank(owner);
        string memory actualPassword = passwordStore.getPassword();
        assertEq(actualPassword, expectedPassword);
    }
```

</details>

**Recommended Mitigation:** Add an access control conditional to the `setPassword` function

```javascript
if(msg.sender != s_owner){
    revert PasswordStore__NotOwner();
}
```

## Likelihood & Impact:

- Impact: HIGH
- Likelihood: HIGH
- Severity: HIGH


## Informational

### [I-1] The `PasswordStore::getPassword` natspec indicates a parameter that doesn't exist, causing the natspec to be incorrect.

**Description:**

    ```javascript
    /*
     * @notice This allows only the owner to retrieve the password.
    @> * @param newPassword The new password to set.
     */
    function getPassword() external view returns (string memory) {}
    ```
​
    The `PasswordStore::getPassword` function signature is `getPassword()` while the NatSpec says it should be `getPassword(string)`.
​
**Impact:** The NatSpec is incorrect.
​
**Recommended Mitigation:** Remove the incorrect NatSpec line.
​
```diff
-     * @param newPassword The new password to set.
```

## Likelihood & Impact:

- Impact: NONE
- Likelihood: HIGH
- Severity: Informational/Gas/Non-Crits

Informational: Hey, this isn't a risk, but you should know...


## Gas
No gas-related findings were identified in this review.