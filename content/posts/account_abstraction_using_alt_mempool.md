---
title: "ERC4337: Account Abstraction Using Alt Mempool"
date: 2023-07-28T10:33:59+08:00
draft: true
tags: ['solidity', 'erc4337', 'account-abstraction', 'learning-note']
summary: An account abstraction proposal which completely avoids consensus-layer protocol changes, instead relying on higher-layer infrastructure.
---

## Overview

- Allow users to use smart contract wallets instead of EOAs as their primary account
    - Free from managing private keys and seed phrases
- Do not require any Ethereum consensus changes
- Allow to pay tx fees with ERC20 tokens

## Specification

### Definitions

Users send `UserOperation` objects to a dedicated user operation mempool (different from traditional transactions triggered by EOAs). **Bundlers** listen in on the user operation mempool, and create **bundle transactions**. A bundle transaction packages up multiple `UserOperation` objects into a single `handleOps` call to a pre-published global **entry point contract**.

- **UserOperation** an ABI-encoded structure that describes a *transaction* to be sent on behalf of a user

    ```js
    struct UserOperation {
        address sender; // the account contract making the operation
        uint256 nonce;  // anti-replay parameter
        bytes initCode; // the initCode of the account (needed if the account is not yet on-chain)
        bytes callData;
        uint256 callGasLimit;
        uint256 verificationGasLimit;
        uint256 preVerificationGas; // the amount of gas to pay for to compensate the bundler for pre-verification execution and calldata
        uint256 maxFeePerGas;
        uint256 maxPriorityFeePerGas;
        bytes paymasterAndData; // address of paymaster sponsoring the transaction, followed by extra data to send to the paymaster (empty for self-sponsored transaction)
        bytes signature;    // data passed into the account along with the nonce during the verification step
    }
    ```

- **EntryPoint** a singleton contract to execute bundles of UserOperations

    ```js
    function handleOps(UserOperation[] calldata ops, address payable beneficiary);

    // can validate multiple ops with one signature
    function handleAggregatedOps(
        UserOpsPerAggregator[] calldata opsPerAggregator,
        address payable beneficiary
    );
    struct UserOpsPerAggregator {
        PackedUserOperation[] userOps;
        IAggregator aggregator;
        bytes signature;
    }
    ```

- **Bundler** a node that can handle UserOperations, create a valid an `EntryPoint.handleOps()` transaction, and add it to the block while it is still valid
- **Aggregator** a helper contract trusted by accounts to validate an aggregated signature

#### Account

```js
interface IAccount {
  function validateUserOp ( // will be called by EntryPoint.handleOps()
    UserOperation calldata userOp,
    bytes32 userOpHash, // a hash over the userOp (except signature), entryPoint and chainId
    uint256 missingAccountFunds
  ) external returns (uint256 validationData);
}
```

The account:

- MUST validate the caller is a trusted EntryPoint
- If the account does not support signature aggregation, it MUST validate the signature is a valid signature of the `userOpHash`, and SHOULD return SIG_VALIDATION_FAILED (and not revert) on signature mismatch
- MUST pay the caller at least the `missingAccountFunds`
- `validationData` a pack of `authorizer`, `validUntil` and `validAfter` timestamps
    - `authorizer` (20 bytes) 0 for valid signature, 1 to mark signature failure. Otherwise, an address of an authorizer (signature aggregator) contract
    - `validUntil` (6 bytes) zero for indefinite
    - `validAfter` (6 bytes) 

## References

- [ERC-4337: Account Abstraction Using Alt Mempool](https://eips.ethereum.org/EIPS/eip-4337)
- [eth-infinitism/account-abstraction](https://github.com/eth-infinitism/account-abstraction/tree/main)
- [Account Abstraction on Ethereum: ERC-4337 Breakdown | LearnWeb3](https://learnweb3.io/lessons/account-abstraction-on-ethereum-erc-4337-breakdown)