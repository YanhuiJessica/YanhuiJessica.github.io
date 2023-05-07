---
title: "EVM Bytecode Programming"
date: 2022-06-13T21:13:40+08:00
draft: false
tags: ['ethereum', 'evm', 'bytecode', 'learning-note']
summary: The basic knowledge of EVM bytecode programming.
---

## Dapp tools installation

- 安装包管理器 `Nix`，重启命令行工具

    ```bash
    $ sh <(curl -L https://nixos.org/nix/install) --daemon
    ```

- 安装 dapptools

    ```bash
    $ curl https://dapp.tools/install | sh
    ```

## Environment setup

- 使用 `dapptools` 启动一个本地测试网络

    ```bash
    dapp testnet
    ```

- 另起一个终端，设置相关环境变量

    ```bash
    export ETH_KEYSTORE=~/.dapp/testnet/8545/keystore
    export ETH_FROM=$(cat ~/.dapp/testnet/8545/config/account | head -n 1)
    export ETH_FROM_2=$(cat ~/.dapp/testnet/8545/config/account | tail -n 1)
    export ETH_PASSWORD=/dev/null
    export ETH_GAS=1000000
    export ETH_RPC_URL=http://localhost:8545
    ```

## The simplest smart contract

- 从最简单的智能合约开始

    ```
    00  // stop
    ```

- 部署合约
  
    ```bash
    $ seth send --create 0x00
    seth-send: Published transaction with 1 bytes of calldata.
    seth-send: 0xcbf0c3c9f0459fef2a4d81730e660ef1190853e93c693d929fb5daa137042ef0
    seth-send: Waiting for transaction receipt...
    seth-send: Transaction included in block 1.
    0x94c09912773862b63f35a7a38ee429873891c1d6

    # 查看合约的字节码，看起来不太对劲...
    $ seth code 0x94c09912773862b63f35a7a38ee429873891c1d6
    0x
    ```

- 为了能够实际部署上合约，需要返回想要部署的合约代码

    ```
    f3  // return
    ```

- `f3` 返回存储在内存中的内容，将需要获取内容的长度以及在内存中的偏移量压入栈中，`60` 为 `push`

    ```
    // 返回自地址 00 的 1 字节内容
    60 01   // length
    60 00   // offset
    f3
    ```

- 使用 `52` 将想返回的内容放入内存中的指定位置

    ```
    // 将 00 放入内存 00 位置
    60 00   // value
    60 00   // offset
    52
    ```

- 完整代码

    ```
    60 00
    60 00
    52    // store 00 in memory position 00
    60 01
    60 00
    f3    // return 1 byte of data from memory position 00
    ```

- 部署合约

    ```bash
    $ seth send --create 600060005260016000f3
    seth-send: Published transaction with 10 bytes of calldata.
    seth-send: 0x577f3091aca11d84f6f2e9f06486bb0bcb91fe7d17761a6d0644b5b7e5ef25f5
    seth-send: Waiting for transaction receipt...
    seth-send: Transaction included in block 3.
    0x2531f6408b6eb475e0b245f2d30ee55ea3bc08a7

    # ΦωΦ 现在看起来没问题了
    $ seth code 0x2531f6408b6eb475e0b245f2d30ee55ea3bc08a7
    0x0
    ```

## Deploy code that returns a string

- 转换想返回的字符串，且与 Solidity 兼容
  - Solidity 的字符串实际上是动态数组

    ```bash
    $ seth --to-uint256 32; # string starts at position 32
    seth --to-uint256 19; # string has 19 characters
    seth --from-ascii 'Hello, EVM bytecode' | seth --to-bytes32
    0x0000000000000000000000000000000000000000000000000000000000000020
    0x0000000000000000000000000000000000000000000000000000000000000013
    0x48656c6c6f2c2045564d2062797465636f646500000000000000000000000000
    ```

- 将字符串放入内存，并返回
  - 字节码 `6i` 一次性将 `i+1` 个字节压入栈中，当长度超过 $16$ 时，使用 `7i` 一次性将 `i+0x11` 个字节压入栈中

    ```
    60 20
    60 00
    52  // put a '32' in memory position '00'
    60 13
    60 20
    52  // put a '19' in memory position '32'
    72 48 65 6c 6c 6f 2c 20 45 56 4d 20 62 79 74 65 63 6f 64 65
    60 40
    52  // put 'Hello, EVM bytecode' in memory position '64'

    // tell f3 the size and position of what we want to return
    60 60
    60 00
    f3  // return it
    ```

- 接下来，编写返回返回字符串 `Hello, EVM bytecode` 代码的代码 XD 简单的做法是使用 `39` 将代码拷贝到内存

    ```
    60 26   // code length
    60 ??   // code position which is gonna be right after this block of code then
    60 00   // memory position
    39
    ```

- 完整代码

    ```
    60 26
    60 0c   // from the first `60 26` to the first `f3`
    60 00
    39
    60 26
    60 00
    f3  // return the code

    60 20
    60 00
    52
    60 13
    60 20
    52
    72 48 65 6c 6c 6f 2c 20 45 56 4d 20 62 79 74 65 63 6f 64 65
    60 40
    52
    60 60
    60 00
    f3 
    ```

- 删除注释，将字节码保存到文件 `simple` 中，部署合约并调用

    ```bash
    $ sed 's/\/\/.*//g' simple | tr '\n' ' ' | sed 's/ //g' | xargs seth send --create
    seth-send: Published transaction with 50 bytes of calldata.
    seth-send: 0x5984a65068b95fbcde25acb088cce10cec5438ca229efdbabf74b3aa8b198b1e
    seth-send: Waiting for transaction receipt...
    seth-send: Transaction included in block 3.
    0x5e2d707a2ea789ec68f26fb9469edd5566adc01d

    $ seth call 0x5e2d707a2ea789ec68f26fb9469edd5566adc01d
    0x000000000000000000000000000000000000000000000000000000000000002000000000000000000000000000000000000000000000000000000000000000130000000000000000000000000048656c6c6f2c2045564d2062797465636f6465

    $ seth call 0x5e2d707a2ea789ec68f26fb9469edd5566adc01d | seth --to-ascii
     Hello, EVM bytecode
    ```

## References

- [Ethereum Virtual Machine Opcodes](https://www.ethervm.io/)
- [EVM bytecode programming - HackMD](https://hackmd.io/@e18r/r1yM3rCCd)
- [dapphub/dapptools](https://github.com/dapphub/dapptools)