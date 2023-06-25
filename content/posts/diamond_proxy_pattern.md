---
title: "Diamond Proxy Pattern"
date: 2023-06-24T15:59:35+08:00
draft: false
tags: ['solidity', 'proxy', 'diamond', 'learning-note']
summary: The basic knowledge of diamond proxy pattern.
---

## Advantages

- 能够解决合约大小限制问题
- 可以使用已部署合约创建 diamonds
- Diamonds 可以是不可变的或可升级的
- 支持合约部分升级

## Terms

- **Diamond** 代理合约
- **Facet** 切面，实现特定功能的单个合约或 library
    - 切面间相互独立，但可以共享内部函数、library 和状态变量
    - 只有包含一个以上外部函数的 library 可以部署并成为切面
- **Loupe facet** 提供 diamond 中使用的切面和函数选择器的信息

## Fallback Function

回调函数根据 `msg.sig` 决定切面地址

```js
// example
// Find facet for function that is called and execute the
// function if a facet is found and return any value.
fallback() external payable {
  // get facet from function selector
  address facet = selectorToFacet[msg.sig];
  require(facet != address(0));
  // Execute external function from facet using delegatecall and return any value.
  assembly {
    // copy function selector and any arguments
    calldatacopy(0, 0, calldatasize())
    // execute function call using the facet
    let result := delegatecall(gas(), facet, 0, calldatasize(), 0, 0)
    // get any return value
    returndatacopy(0, 0, returndatasize())
    // return any return value or error back to the caller
    switch result
      case 0 {revert(0, returndatasize())}
      default {return (0, returndatasize())}
  }
}
```

## Storage

- 切面可通过继承相同的合约或使用相同的 library 来共享内部函数和 library
- 切面可通过在同一存储位置使用相同结构来共享状态变量
- Diamond 和 App 存储模式可混用

### Diamond Storage Pattern

- 切面在结构体中声明状态变量
- 使用唯一字符串或其它数据的哈希确定存储位置，类似于结构体的命名空间

### App Storage Pattern

> AppStorage enforces a **naming/access convention** makes code more readable and prevents name clashes of state variables

- 在一个文件中定义结构体 `AppStorage`，其中包含所有应用相关且计划在切面间共享的状态变量
- 切面中引入 `AppStorage` 结构体并声明一个 `AppStorage` 状态变量 `s`

    ```js
    // example
    import "./AppStorage.sol";

    contract StakingFacet {
        AppStorage internal s;
    }
    ```

## Adding/Replacing/Removing Functions

### `IDiamond` Interface

Diamonds 必须实现 `IDiamond` 接口

```js
interface IDiamond {
    enum FacetCutAction {Add, Replace, Remove}

    struct FacetCut {
        address facetAddress;
        FacetCutAction action;
        bytes4[] functionSelectors;
    }

    // records all function changes to a diamond
    event DiamondCut(FacetCut[] _diamondCut, address _init, bytes _calldata);
}
```

### `IDiamondCut` Interface

- Diamonds 包含函数选择器到切面地址的映射，函数的添加、替换或删除通过修改该映射实现
- 若需要在部署后修改函数选择器的映射，则必须实现 `IDiamondCut` 接口

```js
interface IDiamondCut is IDiamond {
    /// @notice Add/replace/remove any number of functions and optionally execute
    ///         a function with delegatecall
    /// @param _diamondCut Contains the facet addresses and function selectors
    /// @param _init The address of the contract or facet to execute _calldata
    /// @param _calldata A function call, including function selector and arguments
    ///                  _calldata is executed with delegatecall on _init
    function diamondCut(
        FacetCut[] calldata _diamondCut,
        address _init,
        bytes calldata _calldata
    ) external;
}
```

## Inspecting Facets & Functions

Diamonds 通过实现 `IDiamondLoupe` 接口来支持切面和函数自省

### `IDiamondLoupe` Interface

```js
interface IDiamondLoupe {
    struct Facet {
        address facetAddress;
        bytes4[] functionSelectors;
    }

    /// @notice Gets all facet addresses and their four byte function selectors.
    /// @return facets_ Facet
    function facets() external view returns (Facet[] memory facets_);

    /// @notice Gets all the function selectors supported by a specific facet.
    /// @param _facet The facet address.
    /// @return facetFunctionSelectors_
    function facetFunctionSelectors(address _facet) external view returns (bytes4[] memory facetFunctionSelectors_);

    /// @notice Get all the facet addresses used by a diamond.
    /// @return facetAddresses_
    function facetAddresses() external view returns (address[] memory facetAddresses_);

    /// @notice Gets the facet that supports the given selector.
    /// @dev If facet is not found return address(0).
    /// @param _functionSelector The function selector.
    /// @return facetAddress_ The facet address.
    function facetAddress(bytes4 _functionSelector) external view returns (address facetAddress_);
}
```

## References

- [ERC-2535: Diamonds, Multi-Facet Proxy](https://eips.ethereum.org/EIPS/eip-2535)
- [💎 info.diamonds](https://www.info.diamonds/)
- [干货：钻石代理合约最佳安全实践-ODAILY](http://www.odaily.news/post/5187926)