---
title: "Head Overflow Bug in Calldata Tuple ABI-Reencoding"
date: 2022-09-20T23:24:46+08:00
draft: false
tags: ['ethereum', 'compiler-bug', 'learning-note']
summary: ABI-encoding a tuple with a statically-sized calldata array in the last component would corrupt 32 leading bytes of its first dynamically encoded component.
math: true
---

## Affected Version

0.5.8 $\le$ compiler $\le$ 0.8.15

## Affected Contracts

- 当对满足以下条件的 tuple 进行 ABI 重编码时将触发漏洞
  - tuple 的最后一个元素是 statically-sized **calldata** array，数组元素为基本类型 `uint` 或 `bytes32`
  - tuple 包含至少一个动态元素，如 `bytes` 或包含动态数组的结构体
  - 代码使用了 ABI coder v2（自 0.8.0 起默认）

## Technical Details

- tuple 的 ABI 编码包含两部分，静态编码的 *head* 以及动态编码的 *tail*。*head* 中包含静态元素以及动态元素在 *tail* 中的偏移
  
```js
contract D {
    function f(bool a, bytes calldata b, uint[2] calldata c) public pure
        returns (bool, bytes calldata, uint[2] calldata) { 
        bytes32 test = 0x0000000000000000000000000000000000000000000000000000000000000000;
        // The check here will not fail unless the caller actually sets b to zero.
        require (bytes32(b) != test);
        return (a, b, c);
    }
}
```

- 🌰 以参数 `true, 0x00000000000000000000000000000000000000000000000000000000686f6c61, [2, 3]` 调用合约 `D`，参数编码及排列如下

    ```
    0x0000000000000000000000000000000000000000000000000000000000000001 a
    0x0000000000000000000000000000000000000000000000000000000000000080 offset of b
    0x0000000000000000000000000000000000000000000000000000000000000002 c[0]
    0x0000000000000000000000000000000000000000000000000000000000000003 c[1]
    0x0000000000000000000000000000000000000000000000000000000000000020 b (length field)
    0x00000000000000000000000000000000000000000000000000000000686f6c61 b (data field)
    ```

    ```bash
    +---------------------------------------+------------+
    |                  HEAD                 |    TAIL    |
    +------------+-------------+------------+------------+
    | value of a | offset of b | value of c | value of b |
    |    bool    |     uint    | uint256[2] |    bytes   |
    |            |             |            |            |
    +------------+-------------+------------+------------+
    |      1     |      2      |      4     |      3     |
    +------------+-------------+------------+------------+
    # 最下面一行代表编码顺序
    ```

- *tail* 的初 32 字节将被覆盖，Remix 中观察到覆盖值为 $0$

    > The aggressive cleanup combined with the non-linear encoding order of dynamic types resulted in a bug where the initial 32 bytes of the tail could be overwritten during the cleanup of the last head component.

    ```bash
    0x0000000000000000000000000000000000000000000000000000000000000001 a
    0x0000000000000000000000000000000000000000000000000000000000000080 offset of b
    0x0000000000000000000000000000000000000000000000000000000000000002 c[0]
    0x0000000000000000000000000000000000000000000000000000000000000003 c[1]
    0x0000000000000000000000000000000000000000000000000000000000000020 b (length field) # Wrong value
    0x00000000000000000000000000000000000000000000000000000000686f6c61 b (data field)
    ```

    ![Return Value](/img/head_overflow_bug_in_calldata_tuple_abi_reencoding01.jpg)

## References

- [Head Overflow Bug in Calldata Tuple ABI-Reencoding | Solidity Blog](https://blog.soliditylang.org/2022/08/08/calldata-tuple-reencoding-head-overflow-bug/)
- [Layout of State Variables in Storage](https://docs.soliditylang.org/en/v0.8.15/internals/layout_in_storage.html)
- [Layout of Call Data](https://docs.soliditylang.org/en/v0.8.15/internals/layout_in_calldata.html)
- [Contract ABI Specification](https://docs.soliditylang.org/en/v0.8.15/abi-spec.html#)