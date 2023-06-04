---
title: "Gas Optimization Techniques"
date: 2023-06-03T22:46:51+08:00
draft: true
tags: ['solidity', 'gas-optimization', 'learning-note']
---

## Variables

- 减少访问状态变量的次数，有策略地使用本地变量缓存

## Loop

- 在循环判断条件中，避免使用 `>=` 或 `<=`（需要消耗 3 个字节码，`LT` / `GT` + `EQ` + `OR`）
- 在确保不会溢出的情况下，将循环量自增放在 `unchecked` 块中

    ```js
    for (uint i; i < 10;) {
        unchecked {
            i++;
        }
    }
    ```

- 缓存遍历数组的长度

## References

- [Solidity Gas Optimization Techniques: Loops - HackMD](https://hackmd.io/@totomanov/gas-optimization-loops)