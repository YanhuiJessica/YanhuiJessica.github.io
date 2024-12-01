---
title: "Layout of State Variables in Solang Data Account"
date: 2024-11-26T12:15:19+08:00
draft: false
tags: ['solana', 'solang', 'storage', 'learning-note']
summary: Some layout rules for Solang state variables summarized from the documentation and source code.
---

A Solidity contract on Solana utilizes two accounts: a data account and a program account. The data account holds all the storage variables. The layout rules for state variables in data accounts are generally consistent with EVM-compatible contracts, with some exceptions.

The data in the data account starts with a 16-byte header, which stores some metadata. Starting at offset 0, a [4-byte contract magic number](https://github.com/hyperledger-solang/solang/blob/v0.3.3/src/codegen/solana_deploy.rs#L627-L643) is stored, which corresponds to [the first 4 bytes of the keccak256 hash of the contract name](https://github.com/hyperledger-solang/solang/blob/v0.3.3/src/sema/contracts.rs#L55-L63). Starting at offset 12, a [4-byte heap offset](https://github.com/hyperledger-solang/solang/blob/v0.3.3/src/codegen/solana_deploy.rs#L645-L667) is stored in little-endian order[^order], indicating the position where the heap starts in the data account.

The heap offset is the result of rounding up the fixed field size in storage to the nearest multiple of 8. The fixed field contains the static data and the starting offset of each dynamic variable in the storage.

## Dynamic Variables

If the dynamic variable has not been initialized, its starting offset value in the fixed field is 0.

Based on the modification of dynamic variables, the existing chunk in the heap will be modified, or a new chunk will be allocated. Below is the structure of a chunk.

next_chunk_offset (4 bytes) | prev_chunk_offset (4 bytes) | length (4 bytes) | allocated (4 bytes) | data
-|-|-|-|-

Note that the value of the starting offset in the fixed field corresponds to the starting position of the data field in the chunk.

## References

- [Brief Language status — Solang Solidity Compiler v0.3.3 documentation](https://solang.readthedocs.io/en/v0.3.3/language/introduction.html)
- [Data types — Solang Solidity Compiler v0.3.3 documentation](https://solang.readthedocs.io/en/v0.3.3/targets/solana.html#data-types)

[^order]: By default, `SetStorage` uses little-endian. For the contract magic number, the result returned by the hash function is in big-endian, but it is [read in little-endian](https://github.com/hyperledger-solang/solang/blob/v0.3.3/src/sema/contracts.rs#L62) and written in little-endian. Thus, the actual stored value is in big-endian.
