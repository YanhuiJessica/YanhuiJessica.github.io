---
title: "EIP7702: Set EOA Account Code"
date: 2025-04-30T22:24:47+08:00
draft: false
tags: ['solidity', 'account-abstraction', 'learning-note']
summary: Allows EOA to set the delegated code in their account.
---

## Set Code Transaction

- TransactionType: `SET_CODE_TX_TYPE` (0x04)
- TransactionPayload is the RLP serialization of the following:

    ```diff
    rlp([
        chain_id, nonce, max_priority_fee_per_gas,
        max_fee_per_gas, gas_limit, destination,
        value, data, access_list,
    !   authorization_list,
        signature_y_parity, signature_r, signature_s
    ])

    authorization_list = [[chain_id, address, nonce, y_parity, r, s], ...]
    ```

### Behavior

- The authorization list is a list of tuples that indicate what code the signer of each tuple desires to execute in the context of their EOA, and it is processed before the execution portion of the transaction begins, but after the sender’s nonce is incremented.
- For each `[chain_id, address, nonce, y_parity, r, s]` tuple, perform the following:
    1. Verify the chain ID is 0 or the ID of the current chain.
    2. Verify the nonce is less than `2**64 - 1`.
    3. Let `authority = ecrecover(msg, y_parity, r, s)`.
          - Where `msg = keccak(MAGIC || rip([chain_id, address, nonce]))`[^1].
          - Verify `s` is less than or equal to `secp256k1n/2`.
    4. Add `authority` to `accessed_addresses`.
    5. Verify the code of `authority` is empty of already delegated.
    6. Verify the nonce of `authority` is equal to `nonce`.
    7. Add `PER_EMPTY_ACCOUNT_COST - PER_AUTH_BASE_COST` gas to the global refund counter if `authority` is not empty.
    8. **Set the code of `authority` to be `0xef0100 || address` (delegation indicator)**.
          - If `address` is zero address, do not write the delegation indicator. Clear the account's code by reseting the account's code hash to the empty code hash `keccak256("")`.
    9. Increase the nonce of `authority` by one.
- When multiple tuples from the same authority are present, using the address in the last valid occurrence.
- If transaction execution results in failure, the processed delegation indicator is *not rolled back*.

#### Delegation

- The delegation forces all code executing operations to follow the address pointer to obtain the code to execute. Any transaction where `destination` points to an address with a delegation indicator present is affected.
- *During delegated execution `CODESIZE` and `CODECOPY` produce a different result compared to calling `EXTCODESIZE` and `EXTCODECOPY` on the authority*.
    - `EXTCODESIZE` returns `23` (the size of `0xef0100 || address`)
    - `CODESIZE` returns the size of the code residing at `address`.
- When a precompile address is the target of a delegation, the retrieved code is considered empty and instructions targeting this account will execute empty code.
- To avoid possible loops, retrieve only the first code and then stop following the delegation chain.

[^1]: The value of `MAGIC` is `0x05`.
