---
title: "ERC3525: Semi-Fungible Token (SFT)"
date: 2023-07-06T12:51:19+08:00
draft: false
tags: ['solidity', 'erc3525', 'learning-note']
summary: A specification where ERC721 compatible tokens with the same SLOT and different IDs are fungible.
---

## Overview

- **ERC20** Each token is exactly the same as another token
- **ERC721** Each token is unique
- **ERC1155** Each token ID represents a new configurable token type, multi-instance NFT
    - If supply is only 1, treat it as NFT
- **ERC3525** `<ID, SLOT, VALUE>` represents the semi-fungible structure of a token
    - `ID` universally unique
    - `VALUE` similar to the `balance` property of an ERC20 token
    - `SLOT` two tokens with the same slot can be treated as fungible, adding fungibility to the value property of the tokens

## Specification

- Every ERC3525 compliant contract must implement the ERC3525, ERC721 and ERC165 interfaces

    ```js
    interface IERC3525 is IERC165, IERC721 {
        event TransferValue(uint256 indexed _fromTokenId, uint256 indexed _toTokenId, uint256 _value);
        event ApprovalValue(uint256 indexed _tokenId, address indexed _operator, uint256 _value);

        /// @notice when minting or burning
        event SlotChanged(uint256 indexed _tokenId, uint256 indexed _oldSlot, uint256 indexed _newSlot);

        /// @notice Get the number of decimals the token uses for value
        function valueDecimals() external view returns (uint8);
        function balanceOf(uint256 _tokenId) external view returns (uint256);
        function slotOf(uint256 _tokenId) external view returns (uint256);

        function approve(uint256 _tokenId, address _operator, uint256 _value) external payable;
        function allowance(uint256 _tokenId, address _operator) external view returns (uint256);

        /// @notice Transfer value from a specified token to another specified token with the same slot
        function transferFrom(uint256 _fromTokenId, uint256 _toTokenId, uint256 _value) external payable;
        /// @notice Transfer value from a specified token to an address. The receiver contract must implement onERC3525Received()
        function transferFrom(uint256 _fromTokenId, address _to, uint256 _value) external payable returns (uint256);
    }
    ```

- ERC3525 Token Receiver

    ```js
    interface IERC3525Receiver {
        function onERC3525Received(address _operator, uint256 _fromTokenId, uint256 _toTokenId, uint256 _value, bytes calldata _data) external returns (bytes4);
    }
    ```

- `ERC3525SlotEnumerable` allows to publish the full list of `SLOT`s and make them discoverable

    ```js
    interface IERC3525SlotEnumerable is IERC3525 {
        function slotCount() external view returns (uint256);
        function slotByIndex(uint256 _index) external view returns (uint256);
        function tokenSupplyInSlot(uint256 _slot) external view returns (uint256);
        function tokenInSlotByIndex(uint256 _slot, uint256 _index) external view returns (uint256);
    }
    ```

- `ERC3525SlotApprovable` allows to support approval for slots, which allows an operator to manage one's tokens with the same slot

    ```js
    interface IERC3525SlotApprovable is IERC3525 {
        event ApprovalForSlot(address indexed _owner, uint256 indexed _slot, address indexed _operator, bool _approved);

        function isApprovedForSlot(
            address _owner,
            uint256 _slot,
            address _operator
        ) external view returns (bool);

        function setApprovalForSlot(
            address _owner,
            uint256 _slot,
            address _operator,
            bool _approved  // approve or disapprove
        ) external payable;
    }
    ```

## References

- [ERC-3525: Semi-Fungible Token](https://eips.ethereum.org/EIPS/eip-3525)