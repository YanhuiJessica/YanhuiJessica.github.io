---
title: "Namespaced Storage Layout"
date: 2023-12-15T20:36:41+08:00
draft: false
tags: ['solidity', 'storage', 'learning-note']
summary: The basic knowledge of namespaced storage pattern.
---

- 命名空间存储模式已在 [Openzeppelin Contracts 5.0](https://blog.openzeppelin.com/introducing-openzeppelin-contracts-5.0) 中使用 :)

## Specification

- 命名空间是结构体类型
- 根存储位置由 `keccak256(abi.encode(uint256(keccak256(<NAMESPACE_ID>)) - 1)) & ~bytes32(uint256(0xff))` 计算得到

## Example

```solidity
pragma solidity ^0.8.20;

contract Example {
    /// @custom:storage-location erc7201:example.main
    struct MainStorage {
        uint256 x;
        uint256 y;
    }

    // keccak256(abi.encode(uint256(keccak256("example.main")) - 1)) & ~bytes32(uint256(0xff));
    bytes32 private constant MAIN_STORAGE_LOCATION =
        0x183a6125c38840424c4a85fa12bab2ab606c4b6d0e7cc73c0c06ba5300eab500;

    function _getMainStorage() private pure returns (MainStorage storage $) {
        assembly {
            $.slot := MAIN_STORAGE_LOCATION
        }
    }

    function _getXTimesY() internal view returns (uint256) {
        MainStorage storage $ = _getMainStorage();
        return $.x * $.y;
    }
}
```

## References

- [ERC-7201: Namespaced Storage Layout](https://eips.ethereum.org/EIPS/eip-7201)
