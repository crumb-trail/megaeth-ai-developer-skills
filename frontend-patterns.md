# Frontend Patterns (React / Next.js)

## Architecture Principle

**Never open per-user WebSocket connections.** Use one connection, broadcast to users.

```
❌ Wrong: Each user → WebSocket → MegaETH
✅ Right: Server → WebSocket → MegaETH, Server → broadcast → Users
```

## Real-time Data Flow

### Server-Side WebSocket Manager

```typescript
// lib/megaeth-stream.ts
import WebSocket from 'ws';

class MegaETHStream {
  private ws: WebSocket | null = null;
  private subscribers = new Set<(data: any) => void>();
  private keepaliveInterval: NodeJS.Timeout | null = null;

  connect() {
    this.ws = new WebSocket('wss://mainnet.megaeth.com/ws');
    
    this.ws.on('open', () => {
      // Subscribe to mini-blocks
      this.ws!.send(JSON.stringify({
        jsonrpc: '2.0',
        method: 'eth_subscribe',
        params: ['miniBlocks'],
        id: 1
      }));
      
      // Keepalive every 30s
      this.keepaliveInterval = setInterval(() => {
        this.ws!.send(JSON.stringify({
          jsonrpc: '2.0',
          method: 'eth_chainId',
          params: [],
          id: Date.now()
        }));
      }, 30000);
    });

    this.ws.on('message', (data) => {
      const parsed = JSON.parse(data.toString());
      if (parsed.method === 'eth_subscription') {
        this.subscribers.forEach(fn => fn(parsed.params.result));
      }
    });

    this.ws.on('close', () => {
      if (this.keepaliveInterval) clearInterval(this.keepaliveInterval);
      setTimeout(() => this.connect(), 1000); // Reconnect
    });
  }

  subscribe(callback: (data: any) => void) {
    this.subscribers.add(callback);
    return () => this.subscribers.delete(callback);
  }
}

export const megaStream = new MegaETHStream();
```

### Client-Side Hook

```typescript
// hooks/useMiniBlocks.ts
import { useEffect, useState } from 'react';
import { io } from 'socket.io-client';

export function useMiniBlocks() {
  const [latestBlock, setLatestBlock] = useState<MiniBlock | null>(null);
  const [tps, setTps] = useState(0);

  useEffect(() => {
    const socket = io('/api/stream');
    
    socket.on('miniBlock', (block: MiniBlock) => {
      setLatestBlock(block);
      setTps(block.transactions.length * 100); // ~100 mini-blocks/sec
    });

    return () => { socket.disconnect(); };
  }, []);

  return { latestBlock, tps };
}
```

## Transaction Submission

### Optimized Flow

```typescript
// lib/submit-tx.ts
import { createWalletClient, http, custom } from 'viem';
import { megaeth } from './chains';

export async function submitTransaction(signedTx: `0x${string}`) {
  const client = createWalletClient({
    chain: megaeth,
    transport: http('https://mainnet.megaeth.com/rpc')
  });

  // Use sync method for instant receipt
  const receipt = await client.request({
    method: 'eth_sendRawTransactionSync',
    params: [signedTx]
  });

  return receipt; // Receipt available immediately
}
```

### Chain Configuration (viem)

```typescript
// lib/chains.ts
import { defineChain } from 'viem';

export const megaeth = defineChain({
  id: 4326,
  name: 'MegaETH',
  nativeCurrency: { name: 'Ether', symbol: 'ETH', decimals: 18 },
  rpcUrls: {
    default: { http: ['https://mainnet.megaeth.com/rpc'] }
  },
  blockExplorers: {
    default: { name: 'Etherscan', url: 'https://mega.etherscan.io' }
  }
});

export const megaethTestnet = defineChain({
  id: 6343,
  name: 'MegaETH Testnet',
  nativeCurrency: { name: 'Ether', symbol: 'ETH', decimals: 18 },
  rpcUrls: {
    default: { http: ['https://carrot.megaeth.com/rpc'] }
  },
  blockExplorers: {
    default: { name: 'Blockscout', url: 'https://megaeth-testnet-v2.blockscout.com' }
  }
});
```

## Gas Configuration

```typescript
// ❌ Wrong: viem adds 20% buffer
const gasPrice = await publicClient.getGasPrice();

// ✅ Right: use base fee directly
const gasPrice = 1000000n; // 0.001 gwei

// ✅ Right: hardcode gas for known operations
const tx = await walletClient.sendTransaction({
  to: recipient,
  value: amount,
  gas: 21000n,
  maxFeePerGas: 1000000n,
  maxPriorityFeePerGas: 0n
});
```

## RPC Request Batching

```typescript
// ❌ Wrong: Multicall contract
const results = await multicall({ contracts: [...] });

// ✅ Right: JSON-RPC batch
const results = await client.request([
  { method: 'eth_call', params: [call1], id: 1 },
  { method: 'eth_call', params: [call2], id: 2 },
  { method: 'eth_call', params: [call3], id: 3 }
]);

// ❌ Wrong: mixing slow with fast
const results = await client.request([
  { method: 'eth_call', params: [...], id: 1 },     // Fast
  { method: 'eth_getLogs', params: [...], id: 2 }   // Slow - blocks entire batch
]);
```

## Historical Data

Never block UX waiting for historical queries:

```typescript
// Load historical in background
useEffect(() => {
  // Don't await - let it load async
  fetchHistoricalTrades().then(setTrades);
}, []);

// Use indexers for heavy queries
// Recommended: Envio HyperSync
// https://docs.envio.dev/docs/HyperSync/overview
```

## Error Handling

```typescript
const TX_ERRORS = {
  'nonce too low': 'Transaction already executed',
  'already known': 'Transaction pending',
  'intrinsic gas too low': 'Increase gas limit',
  'insufficient funds': 'Not enough ETH for gas'
};

function handleTxError(error: Error) {
  for (const [pattern, message] of Object.entries(TX_ERRORS)) {
    if (error.message.includes(pattern)) {
      return message;
    }
  }
  return 'Transaction failed';
}
```
