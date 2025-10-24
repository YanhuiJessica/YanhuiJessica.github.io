---
title: "[Solidity] Gas Optimization Techniques"
date: 2023-06-03T22:46:51+08:00
draft: true
tags: ['solidity', 'gas', 'gas-optimization', 'learning-note']
summary: Some tested gas optimization techniques for Solidity.
---

## Variables

- 减少访问状态变量的次数，有策略地使用本地变量缓存
- 在条件允许的情况下，选择尺寸小的 uint，从而能够将变量打包在一起减少存储空间
    - 打包同时读写频率高的变量
    - 独立使用频率高的变量，选择 `uint256`，减少 masking operations 的开销

## Loop

- 在循环判断条件中，避免使用 `>=` 或 `<=`（需要消耗 3 个字节码，`LT` / `GT` + `EQ` + `OR`）
- 在确保不会溢出的情况下，将循环量自增放在 `unchecked` 块中（牺牲了可读性）
    - 注：v0.8.22 及之后版本优化了对循环自增量的溢出检查

    ```solidity
    for (uint i; i < 10;) {
        unchecked {
            i++;
        }
    }
    ```

- 缓存遍历数组的长度

## References

- [Solidity Gas Optimization Techniques: Loops - HackMD](https://hackmd.io/@totomanov/gas-optimization-loops)
- [Solidity Gas Golfing #1. Gas Golfing #1: uint8 vs uint256 | by Sudeep Sagar | CoinsBench](https://coinsbench.com/an%CC%A3uha-solidity-gas-golfing-1-6e53269b03e4)
- [The RareSkills Book of Solidity Gas Optimization: 80+ Tips - RareSkills](https://www.rareskills.io/post/gas-optimization)