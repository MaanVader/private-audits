# Introduction

A time-boxed security review of the **luhocoin** protocol was done by **MaanVader**, with a focus on the security aspects of the application's implementation.

# Disclaimer

A smart contract security review can never verify the complete absence of vulnerabilities. This is a time, resource and expertise bound effort where I try to find as many vulnerabilities as possible. I can not guarantee 100% security after the review or even if the review will find any problems with your smart contracts. Subsequent security reviews, bug bounty programs and on-chain monitoring are strongly recommended.

# About **MaanVader**

MaanVader,is an independent smart contract security researcher. Having found numerous security vulnerabilities in various protocols, he does his best to contribute to the blockchain ecosystem and its protocols by putting time and effort into security research & reviews. Reach out on Twitter [@WorstNi43511849](https://twitter.com/WorstNi43511849)



# Severity classification

| Severity               | Impact: High | Impact: Medium | Impact: Low |
| ---------------------- | ------------ | -------------- | ----------- |
| **Likelihood: High**   | Critical     | High           | Medium      |
| **Likelihood: Medium** | High         | Medium         | Low         |
| **Likelihood: Low**    | Medium       | Low            | Low         |

**Impact** - the technical, economic and reputation damage of a successful attack

**Likelihood** - the chance that a particular vulnerability gets discovered and exploited

**Severity** - the overall criticality of the risk

# Security Assessment Summary

**_review commit hash_ - [f7933e647de6a2d8420803556e270236458ca6e9](https://github.com/esk777/luhocoin/blob/master/multiPriceTokenLock.sol)**

### Scope

The following smart contracts were in scope of the audit:

- `multiPriceTokenLock.sol`

The following number of issues were found, categorized by their severity:

- Critical & High: 0 Issues
- Medium: 2 issues
- Low: 2 issues
- Gas Optimization: 5 Issues

---

# Findings Summary

| ID     | Title                   | Severity |
| ------ | ----------------------- | -------- |
| [M-01] |  Uncheked Transfer| Medium |
| [M-02] | Potential Denial-of-Service (DoS) Due to Gas Block Limit    | Medium     |
| [L-01] | Missing `address(0)` checks in the function   | Low   |
| [L-02] | Missing 0 amount check in the `deposit` function | Low      |
| [G-01] | `++i` costs less gas compared to `i++` or `i += 1`                         |   Gas       |
| [G-02] | Initialize variables with no default value                         |    Gas      |
| [G-03] | Public functions not called by the contract should be declared external                         |   Gas       |
| [G-04] | Functions guaranteed to revert when called by normal users can be marked payable                         |    Gas      |
| [G-05] | `x += `y costs more gas than `x = x + y` for state variables                         |    Gas      |

# Medium Severity Issues

# [M-01] Uncheked Transfer

## Severity
**Impact:** Medium, as anyone is able to call the `deposit` and, `withdraw` function can only be called if their addresses are approved

**Likelihood:** Medium, as anyone is able to `deposit` tokens for free but cant `withdraw` as the amounts are checked.

## Description
In the contract `multiPriceTokenLock.sol` the return values for ERC20 `transfer` and `transferFrom` are not checked to be `true`, which could be false if the transferred tokens are not ERC20-Complient. In that case, the transfer fails without being noticed by the calling contract. For example, if one of this tokens are used in the `deposit` function, an attacker could deposit tokens for free.

## Affected Code
* https://github.com/esk777/luhocoin/blob/master/multiPriceTokenLock.sol#L96
* https://github.com/esk777/luhocoin/blob/master/multiPriceTokenLock.sol#L159

## Recommendations
It is recommended to use `SafeERC20` library by  Openzeppelin and call `safeTransfer` or `safeTransferFrom` when transferring ERC20 tokens.

For example:

use:

```solidity

 ERC20Token.safeTransfer(msg.sender, amount);

```

Rather than:

```solidity

 ERC20Token.transfer(msg.sender, amount);
 
```
# [M-02] Potential Denial-of-Service (DoS) Due to Gas Block Limit

## Severity

**Impact:** Medium, as legitimate withdrawals are hindered or delayed.

**Likelihood:** Low, as a large array lenght is required to perform this attack.

## Description

The `withdraw` function iterates through the `payoutTranches` array to check for eligible payouts. However, if the length of the `payoutTranches` array is excessively large, it could exceed the gas block limit and cause the function to fail/revert, resulting in a potential DoS vulnerability.

An attacker could exploit this by deliberately creating or manipulating the `payoutTranches` array to contain a large number of elements, causing the function execution to fail/revert consistently and preventing legitimate users from withdrawing their funds.

## Affected Code
* https://github.com/esk777/luhocoin/blob/master/multiPriceTokenLock.sol#L152-#L157

## Recommendations
To mitigate this issue, you can introduce a loop gas check before proceeding with the next iteration of the loop. Here's an example of a fixed version of the `withdraw` function that includes a gas check:

```solidity
function withdraw() external {
    require(approvedAddresses[msg.sender], "Address not in the approved list");
    require(msg.sender != owner, "Owner cannot withdraw");
    uint256 amount;

    for (uint256 i = 0; i < payoutTranches.length; i++) {
        if (!payoutTranches[i].claimed && payoutTranches[i].activated) {
            payoutTranches[i].claimed = true;
            amount += payoutTranches[i].amount;

            // Gas Check: Check remaining gas and exit the loop if it's close to the gas block limit
            if (gasleft() < GAS_CHECK_THRESHOLD) {
                break;
            }
        }
    }

    require(amount > 0, "No balance to withdraw");
    ERC20Token.safeTransfer(msg.sender, amount);
    emit WithdrawalMade(msg.sender, amount);
}

```
By using `gasleft()` to monitor the remaining gas during each iteration of the loop. If the remaining gas falls below a predetermined threshold (GAS_CHECK_THRESHOLD), the loop is exited, preventing the function from consuming excessive gas and reaching the gas block limit.


# Low Severity Issues

# [L-01] Missing `address(0)` checks in the function

## Description
The `setApprovedAddresses` function allows setting multiple addresses as approved, but it does not include a check to prevent the inclusion of the `address(0)` in the list of approved addresses. This could lead to an unexpected behaviour.

## Affected Code
* https://github.com/esk777/luhocoin/blob/master/multiPriceTokenLock.sol#L46-#L55

## Recommendations
It is advisable to include a check in the setApprovedAddresses function to prevent the inclusion of the address(0). This can be done by adding a condition to validate each address in the addresses array and reject any occurrence of the address(0).

Example:

```solidity

function setApprovedAddresses(address[] calldata addresses) external onlyOwner {
    require(!approvedAddressesLocked, "Approved addresses are locked");

    for (uint256 i = 0; i < addresses.length; i++) {
        // A require statement that checks for address(0)
        require(addresses[i] != address(0), "Invalid address: address(0) not allowed");
        approvedAddresses[addresses[i]] = true;
    }

    emit AddressesApproved();
}

```

# [L-02] Missing 0 amount check in the `deposit` function

## Description
The deposit function allows users to transfer an amount of tokens to the contract. However, it does not include a check to ensure that the amount parameter is greater than zero. This omission could lead to unintended behavior if a user mistakenly or intentionally submits a deposit with an amount of zero. 

## Affected Code
* https://github.com/esk777/luhocoin/blob/master/multiPriceTokenLock.sol#L95-#L99

## Reccomendations
To address this issue, it is recommended to include a check in the `deposit` function to ensure that the amount parameter is greater than zero before proceeding with the deposit. This can be achieved by adding a require statement as follows:

```solidity
function deposit(uint256 amount) external {
    //Add a require statement to check if amount>0
    require(amount > 0, "Amount must be greater than zero");
    ERC20Token.transferFrom(msg.sender, address(this), amount);
    totalDeposits += amount;
    emit DepositMade(msg.sender, amount);
}

```

# Gas Optimization Report
# [G-01] `++i` costs less gas compared to `i++` or `i += 1`
**Total Instances:** `4`

**Instances:**
* https://github.com/esk777/luhocoin/blob/master/multiPriceTokenLock.sol#L50
* https://github.com/esk777/luhocoin/blob/master/multiPriceTokenLock.sol#L64
* https://github.com/esk777/luhocoin/blob/master/multiPriceTokenLock.sol#L129
* https://github.com/esk777/luhocoin/blob/master/multiPriceTokenLock.sol#L152

**Description:**
Pre-increments and pre-decrements are cheaper. This is because post-increments (or post-decrements) return the old value before incrementing or decrementing, hence the name post-increment. On the other hand,pre-increments (or pre-decrements) return the new value. Hence, In the pre-increment case, the compiler has to create a temporary variable (when used) for returning 1 instead of 2.
# [G-02] Initialize variables with no default value
**Total Instances:** `4`

**Instances:**
* https://github.com/esk777/luhocoin/blob/master/multiPriceTokenLock.sol#L50
* https://github.com/esk777/luhocoin/blob/master/multiPriceTokenLock.sol#L64
* https://github.com/esk777/luhocoin/blob/master/multiPriceTokenLock.sol#L129
* https://github.com/esk777/luhocoin/blob/master/multiPriceTokenLock.sol#L152

**Description:**
If a variable is not set/initialized, it is assumed to have the default value (0 for uint, false for bool, address(0) for address…). Explicitly initializing it with its default value is an anti-pattern and wastes gas.

**Example:**

`for (uint256 i = 0; i < num.length; ++i) {};`

Should be replaced with

`for (uint256 i; i < num.length; ++i) {};`
# [G-03] Public functions not called by the contract should be declared external
**Total Instances:** `1`

**Instances:**
* https://github.com/esk777/luhocoin/blob/master/multiPriceTokenLock.sol#L79

**Description:**
Contracts are allowed to override their parents' functions and change the visibility from external to public and can save gas by doing so.
# [G-04] Functions guaranteed to revert when called by normal users can be marked payable
**Total Instances:** `3`

**Instances:**
* https://github.com/esk777/luhocoin/blob/master/multiPriceTokenLock.sol#L48
* https://github.com/esk777/luhocoin/blob/master/multiPriceTokenLock.sol#L59
* https://github.com/esk777/luhocoin/blob/master/multiPriceTokenLock.sol#L90

**Description:** If a function modifier such as onlyOwner is used, the function will revert if a normal user tries to pay the function. Marking the function as payable will lower the gas cost for legitimate callers because the compiler will not include checks for whether a payment was provided. The extra opcodes avoided are CALLVALUE(2),DUP1(3),ISZERO(3),PUSH2(3),JUMPI(10),PUSH1(3),DUP1(3),REVERT(0),JUMPDEST(1),POP(2), which costs an average of about 21 gas per call to the function, in addition to the extra deployment cost.
# [G-05] `x += `y costs more gas than `x = x + y` for state variables
**Total Instances:** `2`

**Instances:**
* https://github.com/esk777/luhocoin/blob/master/multiPriceTokenLock.sol#L97
* https://github.com/esk777/luhocoin/blob/master/multiPriceTokenLock.sol#L155

**Description:** `x += y` costs more than `x = x + y` same as `x -= y`





