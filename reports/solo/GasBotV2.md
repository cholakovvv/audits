# GasbotV2 Security Review

A security review of the **[Gasbot](https://www.gasbot.xyz/)** smart contract protocol was done by **[cholakov](https://twitter.com/cholakovv)**.
This audit report includes all the vulnerabilities, issues and code improvements found during the security review.

## About cholakov

Simeon Cholakov, known as **cholakov**, is an independent smart contract security researcher.
Having found numerous security vulnerabilities in various protocols, he does his
best to contribute to the blockchain ecosystem and its protocols by putting time and
effort into security research & reviews. Check his previous work [here](https://github.com/cholakovvv/audits) or reach out
on Twitter [@cholakovv](https://twitter.com/cholakovv).

## Disclaimer

Audits are a time, resource, and expertise bound effort where trained experts evaluate smart contracts using a combination of automated and manual techniques to identify as many vulnerabilities as possible. Audits can show the presence of vulnerabilities **but not their absence**.

## About Gasbot

Gasbot is a revolutionary service that simplifies gas acquisition across multiple blockchains. It allows you getting gas, without having gas, Instead, if you have stablecoins (like USDC) on any network, you can sign a simple permit message and the protocol will take the specified amount of stable and send you gas on any network.

## Risk classification

| Severity           | Impact: High | Impact: Medium | Impact: Low |
| :----------------- | :----------: | :------------: | :---------: |
| Likelihood: High   |   Critical   |      High      |   Medium    |
| Likelihood: Medium |     High     |     Medium     |     Low     |
| Likelihood: Low    |    Medium    |      Low       |     Low     |

### Impact

- **High** - leads to a significant material loss of assets in the protocol or significantly harms a group of users.
- **Medium** - only a small amount of funds can be lost (such as leakage of value) or a core functionality of the protocol is affected.
- **Low** - can lead to any kind of unexpected behaviour with some of the protocol's functionalities that's not so critical.

### Likelihood

- **High** - attack path is possible with reasonable assumptions that mimic on-chain conditions and the cost of the attack is relatively low to the amount of funds that can be stolen or lost.
- **Medium** - only conditionally incentivized attack vector, but still relatively likely.
- **Low** - has too many or too unlikely assumptions or requires a huge stake by the attacker with little or no incentive.

### Actions required by severity level

- **Critical** - client **must** fix the issue.
- **High** - client **must** fix the issue.
- **Medium** - client **should** fix the issue.
- **Low** - client **could** fix the issue.

## Executive summary

### Overview

|               |                                                                                                                                                       |
| :------------ | :---------------------------------------------------------------------------------------------------------------------------------------------------- |
| Project Name  | Gasbot                                                                                                                                                |
| Repository    | [Link](https://github.com/GasBot-xyz/gasbot_audit)                                                                                                    |
| Commit hash   | [efb5e1d3735f24c7fadb17d59247a262e2647c7b](https://github.com/GasBot-xyz/gasbot_audit/blob/efb5e1d3735f24c7fadb17d59247a262e2647c7b/src/GasbotV2.sol) |
| Documentation | [Link](https://docs.gasbot.xyz/overview/introduction)                                                                                                 |
| Methods       | Manual review                                                                                                                                         |
| Date          | January 3rd, 2024                                                                                                                                     |

### Scope

| File                                                                                                                          | SLOC |
| :---------------------------------------------------------------------------------------------------------------------------- | ---: |
| _Contracts (1)_                                                                                                               |      |
| [src/GasbotV2.sol](https://github.com/GasBot-xyz/gasbot_audit/blob/efb5e1d3735f24c7fadb17d59247a262e2647c7b/src/GasbotV2.sol) |  312 |
| _Abstracts (0)_                                                                                                               |      |
| _Interfaces (0)_                                                                                                              |      |
| _Total (1)_                                                                                                                   |  312 |

### Issues found

| Severity      |  Count |
| :------------ | -----: |
| Critical risk |      0 |
| High risk     |      0 |
| Medium risk   |      3 |
| Low risk      |      7 |
| **Total**     | **10** |

### Summary of Findings

| Severity |   ID   |                                                    Title                                                    |   Status   |
| :------: | :----: | :---------------------------------------------------------------------------------------------------------: | :--------: |
|  Medium  | [M-01] | Not calling approve(0) before setting a new approval causes the call to revert when used with Tether (USDT) | Fixed |
|  Medium  | [M-02] |                              Missing transaction expiration check in `_swap()`                              | Fixed |
|  Medium  | [M-03] |                              Gas griefing is possible on unsafe external call                               | Won't fix |
|   Low    | [L-01] |                                    Consider bounding input array length                                     | Fixed |
|   Low    | [L-02] |                  Consider implementing two-step procedure for updating protocol addresses                   | Fixed |
|   Low    | [L-03] |                         External calls in an un-bounded forloop may result in a DOS                         | Fixed |
|   Low    | [L-04] |           Functions calling contracts/addresses with transfer hooks are missing reentrancy guards           | Fixed |
|   Low    | [L-05] |                                              Loss of precision                                              | Fixed |
|   Low    | [L-06] |                           Unused `receive()` function will lock Ether in contract                           | Fixed |
|   Low    | [L-07] |                                   Return values of approve() not checked                                    | Fixed |

# Detailed findings

## Medium severity

### [M-01] Not calling approve(0) before setting a new approval causes the call to revert when used with Tether (USDT)

#### **Severity**

Medium risk

#### **Context**

- [GasbotV2.sol#L254-L282](https://github.com/GasBot-xyz/gasbot_audit/blob/efb5e1d3735f24c7fadb17d59247a262e2647c7b/src/GasbotV2.sol#L254C4-L282C6)

#### **Description**

Some tokens (like USDT) do not work when changing the allowance from an existing non-zero allowance value. For example Tether (USDT)'s [approve()](https://etherscan.io/token/0xdac17f958d2ee523a2206206994597c13d831ec7#code#L205) function will revert if the current approval is not zero, to protect against front-running changes of approvals.

#### **Recommendation**

It is recommended to use approve(\_router, 0) to set the allowance to zero immediately before each of the existing approve() calls or use OpenZeppelin’s safeApprove/safeIncreaseAllowance.

### [M-02] Missing transaction expiration check in `_swap()`

#### **Severity**

Medium risk

#### **Context**

- [GasbotV2.sol#L268](https://github.com/GasBot-xyz/gasbot_audit/blob/efb5e1d3735f24c7fadb17d59247a262e2647c7b/src/GasbotV2.sol#L268)

#### **Description**

The [\_swap()](https://github.com/GasBot-xyz/gasbot_audit/blob/efb5e1d3735f24c7fadb17d59247a262e2647c7b/src/GasbotV2.sol#L254) function, which is responsible for swapping on Uniswap , sets the deadline argument of the [`exactInput`](https://github.com/GasBot-xyz/gasbot_audit/blob/efb5e1d3735f24c7fadb17d59247a262e2647c7b/src/GasbotV2.sol#L264) for Uniswap V3 call to [`block.timestamp` ](https://github.com/GasBot-xyz/gasbot_audit/blob/efb5e1d3735f24c7fadb17d59247a262e2647c7b/src/GasbotV2.sol#L268) – this basically disables the transaction expiration check because the deadline will be set to whatever timestamp the block including the transaction is minted at.

Transaction expiration check (implemented in Uniswap via the deadline argument) allows users of Uniswap to protect from selling tokens at an outdated price that's lower than the current price.

#### **Recommendation**

Consider a reasonable value to the deadline argument. For example, [Uniswap sets it to 30 minutes on the Ethereum mainnet and to 5 minutes on L2 networks](https://github.com/Uniswap/interface/blob/main/apps/web/src/constants/misc.ts#L9C1-L10C43).

### [M-03] Gas griefing is possible on unsafe external call

#### **Severity**

Medium risk

#### **Context**

- [GasbotV2.sol#L290-L301](https://github.com/GasBot-xyz/gasbot_audit/blob/efb5e1d3735f24c7fadb17d59247a262e2647c7b/src/GasbotV2.sol#L290C5-L301C6)

#### **Description**

`(bool success, )` used in [\_transferAtLeast](https://github.com/GasBot-xyz/gasbot_audit/blob/efb5e1d3735f24c7fadb17d59247a262e2647c7b/src/GasbotV2.sol#L290C5-L301C6) is actually the same as writing `(bool success, bytes memory data)` which basically means that even though the `data` is omitted it doesn’t mean that the contract does not handle it. Actually, the way it works is the `bytes data` that was returned from the `_recipient` will be copied to memory. Memory allocation becomes very costly if the payload is big, so this means that if a `_recipient` implements a fallback function that returns a huge payload, then the `msg.sender` of the transaction, will have to pay a huge amount of gas for copying this payload to memory.

A Malicious actor can launch a gas griefing attack on a relayer. Since griefing attacks have no economic incentive for the attacker and it also requires relayers it should be Medium severity.

#### **Recommendation**

Use a low-level assembly call since it does not automatically copy return data to memory.

```diff
- (bool success, ) = payable(_recipient).call{
-     value: address(this).balance,
-     gas: _gasLimit
- }("");
- require(success, "Transfer failed");

+ bool success;
+ assembly {
+     success := call(_gasLimit, _recipient, address(this).balance, 0, 0, 0, 0)
+ }
```

## Low severity

### [L-01] Consider bounding input array length

#### **Severity**

Low risk

#### **Context**

- [GasbotV2.sol#L467-L470](https://github.com/GasBot-xyz/gasbot_audit/blob/efb5e1d3735f24c7fadb17d59247a262e2647c7b/src/GasbotV2.sol#L467C5-L470C10)

#### **Description**

The [getRelayerBalances](https://github.com/GasBot-xyz/gasbot_audit/blob/efb5e1d3735f24c7fadb17d59247a262e2647c7b/src/GasbotV2.sol#L462C4-L472C6) function take in an unbounded array, and make function calls for entries in the array. While the function will revert if the transaction’s gas cost exceed the block gas limit which means that will be impossible to call this function at all.

#### **Recommendation**

Consider introducing a reasonable upper limit based on block gas limits

### [L-02] Consider implementing two-step procedure for updating protocol addresses

#### **Severity**

Low risk

#### **Context**

- [GasbotV2.sol#L308-L322](https://github.com/GasBot-xyz/gasbot_audit/blob/efb5e1d3735f24c7fadb17d59247a262e2647c7b/src/GasbotV2.sol#L308C5-L322C6)
- [GasbotV2.sol#L328-L336](https://github.com/GasBot-xyz/gasbot_audit/blob/efb5e1d3735f24c7fadb17d59247a262e2647c7b/src/GasbotV2.sol#L328C3-L336C6)

#### **Description**

A copy-paste error or a typo may end up bricking protocol functionality, or sending tokens to an address with no known private key.

#### **Recommendation**

Consider implementing a two-step procedure for updating protocol addresses, where the recipient is set as pending, and must 'accept' the assignment by making an affirmative call. A straight forward way of doing this would be to have the target contracts implement [EIP-165](https://eips.ethereum.org/EIPS/eip-165), and to have the 'set' functions ensure that the recipient is of the right interface type.

### [L-03] External calls in an un-bounded for loop may result in a DOS

#### **Severity**

Low risk

#### **Context**

- [GasbotV2.sol#L454-L458](https://github.com/GasBot-xyz/gasbot_audit/blob/efb5e1d3735f24c7fadb17d59247a262e2647c7b/src/GasbotV2.sol#L454C8-L458C10)

#### **Description**

External calls in an un-bounded for loop may result in a DOS

#### **Recommendation**

Consider limiting the number of iterations in `for` loops that make external calls

### [L-04] Functions calling contracts/addresses with transfer hooks are missing reentrancy guards

#### **Severity**

Low risk

#### **Context**

- [GasbotV2.sol#L420-L423](https://github.com/GasBot-xyz/gasbot_audit/blob/efb5e1d3735f24c7fadb17d59247a262e2647c7b/src/GasbotV2.sol#L420C5-L423C6)

#### **Description**

Adherence to the check-effects-interaction pattern is commendable, but without a reentrancy guard in functions, especially with transfer hooks, users are exposed to read-only reentrancy risks. This can lead to malicious actions without altering the contract state.

#### **Recommendation**

Adding a reentrancy guard is vital for security, protecting against both traditional and read-only reentrancy attacks, ensuring a robust and safe protocol.

### [L-05] Loss of precision

#### **Severity**

Low risk

#### **Context**

- [GasbotV2.sol#L334-L335](https://github.com/GasBot-xyz/gasbot_audit/blob/efb5e1d3735f24c7fadb17d59247a262e2647c7b/src/GasbotV2.sol#L334C13-L335C35)

#### **Description**

Division by large numbers may result in the result being zero, due to solidity not supporting fractions.

#### **Recommendation**

Consider requiring a minimum amount for the numerator to ensure that it is always larger than the denominator.

### [L-06] Unused/empty `receive()` function will lock Ether in contract

#### **Severity**

Low risk

#### **Context**

- [GasbotV2.sol#L507](https://github.com/GasBot-xyz/gasbot_audit/blob/efb5e1d3735f24c7fadb17d59247a262e2647c7b/src/GasbotV2.sol#L507)

#### **Description**

Having no access control on the function means that someone may send Ether to the contract, and have no way to get anything back out, which is a loss of funds.

#### **Recommendation**

If the intention is for the Ether to be used, the function should call another function, otherwise it should revert (e.g. `require(msg.sender == address(weth))`).

### [L-07] Return values of approve() not checked

#### **Severity**

Low risk

#### **Context**

- [GasbotV2.sol#L262](https://github.com/GasBot-xyz/gasbot_audit/blob/efb5e1d3735f24c7fadb17d59247a262e2647c7b/src/GasbotV2.sol#L262)

#### **Description**

Not all IERC20 implementations `revert()` when there's a failure in `approve()`. The function signature has a boolean return value and they indicate errors that way instead. By not checking the return value, operations that should have marked as failed, may potentially go through without actually approving anything

#### **Recommendation**

Use the `safeApprove()` function instead, which reverts the transaction with a proper error message when the return value of `approve` is `false`. A better approach is to use the `safeIncreaseAllowance()` function, which mitigates the multiple withdrawal attack on ERC20 tokens.
