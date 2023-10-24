---
title: "Code4rena - Shell Protocol"
date: 2023-10-11T20:09:01+08:00
draft: false
tags: ['code4rena', 'audit', 'findings-note']
summary: Notes on Shell Protocol findings and analysis report.
---

> [Shell Protocol Findings & Analysis Report](https://code4rena.com/reports/2023-08-shell)

## Overview

- The `EvolvingProteus` contract only contains view functions
- Six external functions: `swapGivenInputAmount()`, `swapGivenOutputAmount()`, `depositGivenInputAmount()`, `depositGivenOutputAmount()`, `withdrawGivenOutputAmount()`, `withdrawGivenInputAmount()`

    ```js
    // Required arguments example
    function swapGivenInputAmount(
        uint256 xBalance,
        uint256 yBalance,   // current balances
        uint256 inputAmount,    // amount
        SpecifiedToken inputToken   // input/output token
    )
    ```

## Lack of Balance Validation

> Invariant violation

The pool's ratio of y to x must be within the interval `[MIN_M, MAX_M)`, which will be checked by the `_checkBalances()` function.

External view functions may call `_swap()`, `_reserveTokenSpecified()` or `_lpTokenSpecified()` functions to get the specified result. However, `_checkBalances()` is only used in the `_swap()` and `_lpTokenSpecified()` functions. There is no balance validation for `depositGivenInputAmount()` and `withdrawGivenOutputAmount()` functions, which use `_reserveTokenSpecified()` function.

If there's no other validation outside these two functions, user deposits/withdraws may break the invariant, i.e. the pool's ratio of y to x is outside the interval `[MIN_M, MAX_M)`.

## Low Risk and Non-Critical Issues

- There are some issues in the comments, which have been informed in the discord channel, but still included in the selected report
