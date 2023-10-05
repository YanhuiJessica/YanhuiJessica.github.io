---
title: "Ethereum Block Header: logsBloom Field"
date: 2023-10-04T19:46:43+08:00
draft: true
math: true
tags: ['ethereum', 'block', 'bloom-filter', 'log', 'learning-note']
summary: Introducing the Bloom filter function.
---

**logsBloom** (2048 bits) The Bloom filter composed from indexable information (logger address and log topics). It's used to efficiently check if a log event from a contract execution is included in the block.

## Bloom filter function

$M(O)\equiv \bigvee_{x\in \{O_a\}\cup O_t}(M_{3:2048}(x))$

- $O_a$ is a tuple of the logger's address
- $O_t$ a possibly empty series of 32-byte log topics
- $M_{3:2048}$ is a specialised Bloom filter that sets three bits out of 2048

### Example

> event Transfer(address indexed from, address indexed to, uint256)

```bash
contract    0x72877813ddc9BcAacD9760B3e76D07732Af369F0
from        0x1061f53BE776fc9E3727705528EF5A6731ABB07d
to          0x0000000000000000000000000000000000001337
topic       0xddf252ad1be2c89b69c2b068fc378daa952ba7f163c4a11628f55a4df523b3ef
```

Indexed arguments are treated like additional topics.

Steps | Description | Values
-|-|-
0| Keccak256(Topic) | ada389e1fc24a8587c776340efb91b36e675792ab631816100d55df0b5cf3cbc
1| First 3 pairs of hash | ada3,89e1,fc24
2| Modulo by 2048 | 1443,481,1060

Same steps for the *contract* address, *from* and *to*.

||Keccak256|Bit Index
-|-|-
contract (no padding)|65c646fc7094c5517d29435e58cd7f4c2aa9a6d3c86799a07ea37dc573ea138c|1478,1788,148
from (padded to 32 bytes)|ec795b8ef819df2699ee2f96301a884c110d89242f4a99b02ad96c5760557fa7|1145,910,25
to (padded to 32 bytes)|7c75ac776ffe94a5fe65ac39474f454d6dca094fc4e5a476898164cda3f4aa25|1141,1143,2046

#### logsBloom

2047|2046|...|1788|...|1478|...|1443|...|1145|1144|1143|1142|1141|...|1060|...|910|...|481|...|148|...|25|...|0
-|-|-|-|-|-|-|-|-|-|-|-|-|-|-|-|-|-|-|-|-|-|-|-|-|-|
0|1|0|1|0|1|0|1|0|1|0|1|0|1|0|1|0|1|0|1|0|1|0|1|0|0

## References

- [ETH Goes Bloom; Filling up Ethereum’s Bloom Filters | by Nate Rush | Medium](https://medium.com/@naterush1997/eth-goes-bloom-filling-up-ethereums-bloom-filters-68d4ce237009)
- [Understanding event logs on the Ethereum blockchain | by Luit Hollander | MyCrypto | Medium](https://medium.com/mycrypto/understanding-event-logs-on-the-ethereum-blockchain-f4ae7ba50378)
- [Yul — Solidity documentation](https://docs.soliditylang.org/en/latest/yul.html)
