---
title: "Bug Concerning Data Location During Inheritance"
date: 2023-10-22T19:33:50+08:00
draft: false
tags: ['ethereum', 'compiler-bug', 'learning-note']
summary: When calling external functions, it is irrelevant if the data location of the parameters is `calldata` or `memory`, the encoding of the data does not change. Thus, changing the data location when overriding external functions is allowed. Since public functions can be called internally as well as externally, this causes invalid code to be generated when such an incorrectly overridden function is called internally through the base contract. The caller provides a memory pointer, but the called function interprets it as a calldata pointer or vice-versa.
math: true
---

## Affected Version

0.6.9 $\le$ compiler $\le$ 0.8.14

## Affected Contracts

- 当派生合约中的函数满足以下条件时将触发漏洞
  - 继承时改变了参数的存储位置
  - 存在对该函数的内部调用，但基于基类合约中原始的函数签名

## Technical Details

以下示例，漏洞发生在合约 `X` 中。

```solidity
abstract contract I {
    // The base contract uses "calldata"
    function f(uint[] calldata x) virtual internal;
}
contract C is I {
    // The derived contract uses "memory" and the compiler
    // does not complain - this is the bug in the compiler.
    function f(uint[] memory x) override internal {
        // If you use `x`, it will access memory
        // even if `D.g` passed us a calldata pointer.
        // This leads to the wrong data being accessed.
    }
}
abstract contract D is I {
    function g(uint[] calldata x) external {
        // Since D only "knows" `I`, the signature of `f`
        // uses calldata, while the virtual lookup ends
        // up with `C.f`, which uses memory. This results
        // in the calldata pointer `x` being passed and
        // interpreted as a memory pointer.
        f(x);
    }
}
contract X is C, D { }
```

有趣的是，以上示例中，`f()` 调用结束的跳转地址为 `x.offset (0x4 + CALLDATA[0x4]) + 0x20` 。

## References

- [Bug Concerning Data Location during Inheritance | Solidity Programming Language](https://soliditylang.org/blog/2022/05/17/data-location-inheritance-bug/)