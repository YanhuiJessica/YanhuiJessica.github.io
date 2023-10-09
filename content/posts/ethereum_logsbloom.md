---
title: "Ethereum Block Header: logsBloom Field"
date: 2023-10-04T19:46:43+08:00
draft: false
math: true
tags: ['ethereum', 'block', 'bloom-filter', 'logsBloom', 'log', 'learning-note']
summary: Introducing the Bloom filter function and explaining how to produce desired logsBloom.
---

**logsBloom** (2048 bits) The Bloom filter composed from indexable information (logger address and log topics). It's used to efficiently check if a log event from a contract execution is included in the block.

## Bloom filter function

$M(O)\equiv \bigvee_{x\in \{O_a\}\cup O_t}(M_{3:2048}(x))$

- $O_a$ is a tuple of the logger's address
- $O_t$ a possibly empty series of 32-byte log topics
- $M_{3:2048}$ is a specialized Bloom filter that sets three bits out of 2048

$H_b\equiv \bigvee_{r\in B_R}(r_b)$

The block-level logsBloom $H_b$ combines the logsBloom of each transaction.

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

For a particular topic, if all relevant bits are set in the Bloom filter, then it's probably present, otherwise it's definitely not present.

## Produce desired logsBloom

Use Yul `log` instruction to log without emitting exact events.

```py
from Crypto.Util.number import getPrime, getRandomNBitInteger
from web3 import Web3
from cheb3 import Connection
from cheb3.utils import calc_create_address, compile_sol

def get_bit_index(v):
    h = Web3.solidity_keccak(["bytes"], [bytes.fromhex(v)])
    idx = []
    for i in range(3):
        idx.append(int.from_bytes(h[i * 2: (i + 1) * 2], "big") % 2048)
    return idx

conn = Connection("http://localhost:8545")
account = conn.account()
nonce = 0

p = getPrime(1024)
topics = []
filled_bits = set()

contract_addr = calc_create_address(account.address, nonce)
while True:
    q = getPrime(1024)
    n = p * q
    bits = get_bit_index(contract_addr[2:])
    if (n & (1 << bits[0])) and (n & (1 << bits[1])) and (n & (1 << bits[2])):
        filled_bits = filled_bits.union(set(bits))
        break

required_bits = bin(n).count("1")
required_hits = 3
print(f"n: {hex(n)}")
print(f"required_bits: {required_bits}")

while len(filled_bits) < required_bits:
    topic = getRandomNBitInteger(256)   # get a random topic
    bits = get_bit_index(f"{topic:064x}")
    if (n & (1 << bits[0])) and (n & (1 << bits[1])) and (n & (1 << bits[2])):
        if len(set(bits).intersection(filled_bits)) <= 3 - required_hits:
            filled_bits = filled_bits.union(set(bits))
            if len(filled_bits) > required_bits - required_bits // 40:	# set as per preference
                required_hits = 1
            topics.append(topic)

if len(topics) % 4 != 0:
    topics.extend([topics[-1]] * (4 - len(topics) % 4))

source = '''
contract Logger {
    function Log(bytes calldata data) external {
        uint256 l = data.length / 32;
        assembly {
            for {let i := 0} lt(i, l) { i := add(i, 4) } {
                let p := add(0x44, mul(i, 0x20))
                log4(
                    0x0,
                    0x0,
                    calldataload(p),
                    calldataload(add(p, 0x20)),
                    calldataload(add(p, 0x40)),
                    calldataload(add(p, 0x60))
                )
            }
        }
    }
}
'''
log_abi, log_bin = compile_sol(source, solc_version="0.8.19")["Logger"]

logger = conn.contract(account, abi=log_abi, bytecode=log_bin)
logger.deploy()

data = b"".join([t.to_bytes(32, "big") for t in topics])
logger.functions.Log(data).send_transaction()
print("logsBloom:", conn.w3.eth.get_block("latest").logsBloom.hex())
```

## References

- [ETH Goes Bloom; Filling up Ethereum’s Bloom Filters | by Nate Rush | Medium](https://medium.com/@naterush1997/eth-goes-bloom-filling-up-ethereums-bloom-filters-68d4ce237009)
- [Ethereum Yellow Paper](https://ethereum.github.io/yellowpaper/paper.pdf#Hfootnote.3)
- [Understanding event logs on the Ethereum blockchain | by Luit Hollander | MyCrypto | Medium](https://medium.com/mycrypto/understanding-event-logs-on-the-ethereum-blockchain-f4ae7ba50378)
- [Yul — Solidity documentation](https://docs.soliditylang.org/en/latest/yul.html)
