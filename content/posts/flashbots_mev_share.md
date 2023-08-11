---
title: "Flashbots MEV-Share"
date: 2023-08-11T10:15:11+08:00
draft: false
tags: ['flashbots', 'mev-share', 'python', 'learning-note']
summary: Python scripts about using flashbots MEV-Share.
---

## Getting Event Steam

```py
import requests
import json

STREAM_ENDPOINT = "https://mev-share-goerli.flashbots.net"

r = requests.get(STREAM_ENDPOINT, stream=True)

for line in r.iter_lines():
    if line and line != b':ping':
        tx = json.loads(line[5:].decode())
        # do sth
```

### Event Example

> Each event property is optional. Transaction senders can choose not to share the information.

Param | Description
-|-
`hash` | Transaction hash
`logs` | Event logs emitted by executing the transaction
`txs` | Transactions from the event. Will only be one if event is a transaction, otherwise event is a bundle

```bash
data: {"hash":"0x9f9631277562383067dd4a87601d4eedc4f7371e7f1bf40ad31bd8dcbf022809","logs":null,"txs":null,"mevGasPrice":"0x1e0a6e0","gasUsed":"0x75300"}

data: {"hash":"0xd202b7c5b71aa6c4f71e4722701d92d9b82aa99e9eb6c2f3af883fda329bcd82","logs":[{"address":"0x118bcb654d9a7006437895b51b5cd4946bf6cdc2","topics":["0x86a27c2047f889fafe51029e28e24f466422abe8a82c0c27de4683dda79a0b5d"],"data":"0x000000000000000000000000000000000000000000000000000e07e5d05d9995000000000000000000000000000000000000000000000000000e07e5d05d99bd"}],"txs":null,"mevGasPrice":"0x2faf080","gasUsed":"0x8ca0"}

data: {"hash":"0x417da3313b21c05bad4e63835f35c9dd7ea1173380e9ce49a6ffe6e3f8875da7","logs":null,"txs":[{"to":"0x1cddb0ba9265bb3098982238637c2872b7d12474","functionSelector":"0xa3c356e4","callData":"0xa3c356e4"}],"mevGasPrice":"0x2faf080","gasUsed":"0x7530"}
```

## Sending Bundles

A bundle is **an ordered array of transactions** that execute atomically.

For example, we want to backrun a transaction similar to the following:

```bash
data: {"hash":"0x417da3313b21c05bad4e63835f35c9dd7ea1173380e9ce49a6ffe6e3f8875da7","logs":null,"txs":[{"to":"0x1cddb0ba9265bb3098982238637c2872b7d12474","functionSelector":"0xa3c356e4","callData":"0xa3c356e4"}],"mevGasPrice":"0x2faf080","gasUsed":"0x7530"}
```

```py
from cheb3 import Connection
from web3 import Web3
from eth_account import Account, messages
import os
import requests
import json

STREAM_ENDPOINT = "https://mev-share-goerli.flashbots.net"
BUNDLE_RELAY_URL = "https://relay-goerli.flashbots.net"
CHAINID = 5
RPC_URL = os.getenv("RPC_URL", default="")
PRIVATE_KEY = os.getenv("PRIVATE_KEY", default="")

NUM_TARGET_BLOCKS = 30

conn = Connection(RPC_URL)
auth = conn.w3.eth.account.create()	# for flashbots authentication
signer = conn.account(PRIVATE_KEY).eth_acct	# used to sign txs

r = requests.get(STREAM_ENDPOINT, stream=True)

for line in r.iter_lines():
    if line and line != b':ping':
        tx = json.loads(line[5:].decode())
        if tx['txs']:
            signed_tx = signer.sign_transaction({
                'nonce': conn.w3.eth.get_transaction_count(signer.address),
                'gasPrice': conn.w3.eth.gas_price,
                'gas': 400000,
                'to': Web3.to_checksum_address(tx['txs'][0]['to']),
                'from': signer.address,
                'chainId': CHAINID,
                'value': 0,
                'data': '0xb88a802f',
            }).rawTransaction
            bundle = [
                {'hash': tx['hash']},
                {
                    'tx': signed_tx.hex(),
                    'canRevert': False,
                }
            ]

            target_block = conn.w3.eth.get_block('latest')['number'] + 1
            data = {
                'params': [
                    {
                        'version': 'v0.1',
                        'inclusion': {
                            'block': hex(target_block), # block number must be in hex string
                            'maxBlock': hex(target_block + NUM_TARGET_BLOCKS),
                        },
                        'body': bundle,
                    }
                ],
                'jsonrpc': '2.0',
                'id': 1,
                'method': 'mev_sendBundle',
            }
            json_data = json.dumps(data, separators=(',', ':'))
            msg = messages.encode_defunct(text=Web3.keccak(text=json_data).hex())
            sig = Account.sign_message(msg, auth._private_key).signature.hex()
            resp = requests.post(BUNDLE_RELAY_URL, data=json_data, headers={
                'Content-Type': 'application/json',
                'X-Flashbots-Signature': f'{auth.address}:{sig}'
            })
            print(resp.text)
```

## References

- [RPC Endpoint | Flashbots Docs](https://docs.flashbots.net/flashbots-auction/searchers/advanced/rpc-endpoint)
- [Event Stream | Flashbots Docs](https://docs.flashbots.net/flashbots-mev-share/searchers/event-stream)
- [Sending Bundles | Flashbots Docs](https://docs.flashbots.net/flashbots-mev-share/searchers/sending-bundles)