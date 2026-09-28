# Ethereum JSON-RPC: From Chain Head to Contract State

Ethereum applications talk to the execution layer through JSON-RPC. This guide starts with raw RPC calls so you can see what a wallet, SDK, indexer, or backend is actually asking a node to do.

The examples use a public Ethereum endpoint from OnFinality:

```bash
export ETH_RPC=https://eth.api.onfinality.io/public
```

No API key is required for the examples below.

## The useful mental model: gossip, state, and history

Ethereum's own JSON-RPC documentation groups many common methods into three practical buckets:

- **Gossip**: follow the head of the chain and submit transactions.
- **State**: ask what the EVM state looks like at a particular block.
- **History**: retrieve blocks, transactions, receipts, and logs that already exist.

That distinction helps when debugging RPC problems. A node may be perfectly healthy for current-state reads while a deep historical-state query still requires archive data.

## 1. Confirm that you are on Ethereum Mainnet

Ethereum Mainnet has chain ID `1`.

```bash
curl -s "$ETH_RPC" \
  -H 'content-type: application/json' \
  --data '{
    "jsonrpc":"2.0",
    "id":1,
    "method":"eth_chainId",
    "params":[]
  }'
```

The result should be:

```json
{"jsonrpc":"2.0","id":1,"result":"0x1"}
```

Ethereum JSON-RPC encodes quantities as compact hexadecimal values. `0x1` is decimal `1`; `0x10` is decimal `16`.

## 2. Follow the chain head

`eth_blockNumber` returns the latest block number known to the node.

```bash
curl -s "$ETH_RPC" \
  -H 'content-type: application/json' \
  --data '{"jsonrpc":"2.0","id":1,"method":"eth_blockNumber","params":[]}'
```

To fetch the actual block rather than just its height:

```bash
curl -s "$ETH_RPC" \
  -H 'content-type: application/json' \
  --data '{
    "jsonrpc":"2.0",
    "id":1,
    "method":"eth_getBlockByNumber",
    "params":["latest",false]
  }'
```

The second parameter controls whether the block contains full transaction objects (`true`) or only transaction hashes (`false`).

## 3. Read Ethereum state at a block

A balance is state, so the request includes a block tag.

```bash
curl -s "$ETH_RPC" \
  -H 'content-type: application/json' \
  --data '{
    "jsonrpc":"2.0",
    "id":1,
    "method":"eth_getBalance",
    "params":["0xYOUR_ADDRESS","latest"]
  }'
```

The returned value is denominated in wei and encoded as hex.

For applications that care about stronger settlement guarantees, execution clients also understand tags such as `safe` and `finalized` where supported by the node/client combination.

## 4. Call a contract without sending a transaction

`eth_call` executes EVM code locally against a selected block state. It does not publish a transaction and does not consume gas onchain.

```bash
curl -s "$ETH_RPC" \
  -H 'content-type: application/json' \
  --data '{
    "jsonrpc":"2.0",
    "id":1,
    "method":"eth_call",
    "params":[
      {
        "to":"0xCONTRACT_ADDRESS",
        "data":"0xABI_ENCODED_CALLDATA"
      },
      "latest"
    ]
  }'
```

In production code, ABI-aware libraries such as viem or ethers are usually easier than constructing calldata by hand.

## 5. Work with transaction history

Fetch a transaction:

```bash
curl -s "$ETH_RPC" \
  -H 'content-type: application/json' \
  --data '{
    "jsonrpc":"2.0",
    "id":1,
    "method":"eth_getTransactionByHash",
    "params":["0xTRANSACTION_HASH"]
  }'
```

Then fetch its receipt:

```bash
curl -s "$ETH_RPC" \
  -H 'content-type: application/json' \
  --data '{
    "jsonrpc":"2.0",
    "id":1,
    "method":"eth_getTransactionReceipt",
    "params":["0xTRANSACTION_HASH"]
  }'
```

The receipt is usually the object you want when checking whether execution succeeded. Important fields include `status`, `blockNumber`, `gasUsed`, and `logs`.

A `null` receipt normally means the transaction has not been included yet, the hash is wrong, or the node does not know the transaction.

## 6. Query contract events

Logs are emitted by contracts and stored in transaction receipts. `eth_getLogs` lets you scan them directly:

```bash
curl -s "$ETH_RPC" \
  -H 'content-type: application/json' \
  --data '{
    "jsonrpc":"2.0",
    "id":1,
    "method":"eth_getLogs",
    "params":[{
      "fromBlock":"0xSTART_BLOCK",
      "toBlock":"0xEND_BLOCK",
      "address":"0xCONTRACT_ADDRESS"
    }]
  }'
```

For an indexer, avoid asking for an enormous range in one request. Process bounded windows and persist the last successfully indexed block.

## 7. Estimate first, sign locally, broadcast last

Before building a transaction, estimate its execution gas:

```bash
curl -s "$ETH_RPC" \
  -H 'content-type: application/json' \
  --data '{
    "jsonrpc":"2.0",
    "id":1,
    "method":"eth_estimateGas",
    "params":[{
      "from":"0xYOUR_ADDRESS",
      "to":"0xDESTINATION",
      "value":"0x0",
      "data":"0x"
    }]
  }'
```

A normal application signs the transaction locally, then sends only the signed bytes:

```json
{
  "jsonrpc": "2.0",
  "id": 1,
  "method": "eth_sendRawTransaction",
  "params": ["0xSIGNED_TRANSACTION"]
}
```

Never send a private key to an RPC endpoint.

## Minimal JavaScript RPC client

Node 18+ has `fetch`, so no dependency is required:

```js
const rpcUrl = 'https://eth.api.onfinality.io/public';
let id = 0;

async function rpc(method, params = []) {
  const response = await fetch(rpcUrl, {
    method: 'POST',
    headers: {'content-type': 'application/json'},
    body: JSON.stringify({jsonrpc: '2.0', id: ++id, method, params}),
  });

  const body = await response.json();
  if (body.error) throw new Error(`${body.error.code}: ${body.error.message}`);
  return body.result;
}

const chainId = Number.parseInt(await rpc('eth_chainId'), 16);
const blockNumber = Number.parseInt(await rpc('eth_blockNumber'), 16);

console.log({chainId, blockNumber});
```

## Common Ethereum RPC gotchas

### Hex quantities are not normal decimal strings

Block numbers, balances, nonces, gas values, and many other numeric fields are returned as hex quantities. Convert them deliberately rather than assuming decimal.

### Historical block access is not the same as historical state access

A node can retain old block bodies while pruning old state tries. Queries such as an old `eth_getBalance(..., blockNumber)` may therefore require archive access even when `eth_getBlockByNumber` for the same block works.

### JSON-RPC is the execution API, not the Beacon API

Ethereum consensus-layer data is exposed through the Beacon API. The JSON-RPC calls in this repository target the execution layer.

## References

- [Ethereum JSON-RPC API](https://ethereum.org/en/developers/docs/apis/json-rpc/)
- [OnFinality Ethereum RPC](https://onfinality.io/en/networks/eth)
- [OnFinality network directory](https://onfinality.io/en/networks)
