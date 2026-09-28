# Ethereum RPC Getting Started

A practical introduction to calling Ethereum Mainnet over JSON-RPC. The examples use a public OnFinality endpoint so you can run them immediately without creating an account.

## RPC endpoint

```text
https://eth.api.onfinality.io/public
```

Ethereum uses JSON-RPC 2.0 over HTTP.

## 1. Verify the network

```bash
curl -s https://eth.api.onfinality.io/public \
  -H 'content-type: application/json' \
  --data '{"jsonrpc":"2.0","id":1,"method":"eth_chainId","params":[]}'
```

Ethereum Mainnet has chain ID `1`, returned as `0x1`.

## 2. Get the latest block

```bash
curl -s https://eth.api.onfinality.io/public \
  -H 'content-type: application/json' \
  --data '{"jsonrpc":"2.0","id":1,"method":"eth_blockNumber","params":[]}'
```

The result is hexadecimal. Convert it with:

```js
const block = Number.parseInt('0x1234', 16);
```

To fetch a block object:

```bash
curl -s https://eth.api.onfinality.io/public \
  -H 'content-type: application/json' \
  --data '{"jsonrpc":"2.0","id":1,"method":"eth_getBlockByNumber","params":["latest",false]}'
```

## 3. Read an ETH balance

```bash
curl -s https://eth.api.onfinality.io/public \
  -H 'content-type: application/json' \
  --data '{
    "jsonrpc":"2.0",
    "id":1,
    "method":"eth_getBalance",
    "params":["0x0000000000000000000000000000000000000000","latest"]
  }'
```

The result is a hex-encoded wei value.

## 4. Call a smart contract

Use `eth_call` for read-only contract calls:

```bash
curl -s https://eth.api.onfinality.io/public \
  -H 'content-type: application/json' \
  --data '{
    "jsonrpc":"2.0",
    "id":1,
    "method":"eth_call",
    "params":[
      {"to":"0xYOUR_CONTRACT_ADDRESS","data":"0xYOUR_ABI_ENCODED_CALLDATA"},
      "latest"
    ]
  }'
```

Use an ABI-aware library such as ethers or viem in real applications instead of manually encoding calldata.

## 5. Use the endpoint from JavaScript

```js
const RPC_URL = 'https://eth.api.onfinality.io/public';

async function rpc(method, params = []) {
  const res = await fetch(RPC_URL, {
    method: 'POST',
    headers: {'content-type': 'application/json'},
    body: JSON.stringify({jsonrpc: '2.0', id: 1, method, params}),
  });

  const body = await res.json();
  if (body.error) throw new Error(body.error.message);
  return body.result;
}

const chainId = Number.parseInt(await rpc('eth_chainId'), 16);
const block = Number.parseInt(await rpc('eth_blockNumber'), 16);
console.log({chainId, block});
```

## Useful methods

| Method | Use |
| --- | --- |
| `eth_getTransactionByHash` | Fetch a transaction |
| `eth_getTransactionReceipt` | Check execution status and logs |
| `eth_getCode` | Check contract bytecode |
| `eth_getLogs` | Query contract events |
| `eth_estimateGas` | Estimate gas |
| `eth_feeHistory` | Inspect recent fee data |
| `eth_sendRawTransaction` | Broadcast a signed transaction |

## Troubleshooting

### `429 Too Many Requests`

Public endpoints are shared. Reduce concurrency, cache repeated reads, use retry/backoff logic, or move sustained traffic to an authenticated endpoint.

### `execution reverted`

This usually comes from EVM execution rather than the HTTP connection. Verify the contract address, calldata, caller assumptions, and block tag.

### Historical state is missing

Very old state queries may require archive access. Historical blocks and historical state are not the same capability.

## Mainnet settings

| Setting | Value |
| --- | --- |
| Network | Ethereum Mainnet |
| Chain ID | `1` |
| Native token | ETH |
| RPC | `https://eth.api.onfinality.io/public` |
| WebSocket | `wss://eth.api.onfinality.io/public-ws` |

## Resources

- [Ethereum JSON-RPC documentation](https://ethereum.org/en/developers/apis/json-rpc/)
- [OnFinality Ethereum RPC](https://onfinality.io/en/networks/eth)
- [OnFinality RPC network directory](https://onfinality.io/en/networks)
