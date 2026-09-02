---
name: injective-funding
description: Mass fund Injective wallets with INJ, USDT, or USDC. Batch bank transfers (MsgSend), deposit to exchange subaccounts, top up taker wallets, and check balances across many wallets. Supports batching 200+ transfers in a single transaction. Also covers public-facing faucet servers that accept caller-supplied addresses in either inj1 or 0x form.
license: MIT
metadata:
  author: ck
  version: "1.1.0"
---

# Injective Mass Funding Skill

## Overview

Fund many Injective wallets efficiently. Batch bank transfers, deposit to exchange subaccounts, and verify balances using self-contained `pyinjective` / TypeScript routines.

## Core Funding Patterns

### 1. Batch Bank Transfers (200 Transfers in One Tx)
Batch up to 200 `MsgSend` messages in a single transaction:
```python
from decimal import Decimal
from pyinjective.composer import Composer
from pyinjective.core.broadcaster import MsgBroadcasterWithPk
from pyinjective.core.network import Network

network = Network.testnet()
composer = Composer(network=network.string())
broadcaster = MsgBroadcasterWithPk.new_using_simulation(
    network=network,
    private_key=funder_private_key_hex
)

msgs = []
for wallet in wallets:
    msgs.append(composer.msg_send(
        sender=funder_address,
        receiver=wallet["inj_address"],
        amount=Decimal("10.0"),
        denom="inj"
    ))

# Broadcast batched transaction
tx_response = await broadcaster.broadcast(msgs)
print(f"Tx Hash: {tx_response['txhash']}")
```

### 2. Deposit to Exchange Subaccount
Move funded tokens from a bank address to a trading subaccount:
```python
from pyinjective.composer import Composer

# Subaccount 0 is address + 24 zeros
subaccount_id = f"{funder_eth_address.lower()}000000000000000000000000"

msg = composer.msg_subaccount_deposit(
    sender=funder_address,
    subaccount_id=subaccount_id,
    amount=Decimal("5.0"),
    denom="inj"
)
await broadcaster.broadcast([msg])
```

### 3. Public-Facing Faucet Server Pattern
A secure, minimal Express server that accepts addresses in either `inj1` or `0x` form:
```ts
import express from 'express';
import { Network } from '@injectivelabs/networks';
import {
  MsgBroadcasterWithPk,
  MsgSend,
  PrivateKey,
  getEthereumAddress,
  getInjectiveAddress
} from '@injectivelabs/sdk-ts';

const app = express();
app.use(express.json());

const INJ_BECH32 = /^inj1[02-9ac-hj-np-z]{38}$/;
const ETH_HEX    = /^0x[0-9a-fA-F]{40}$/;

function normalizeAddress(raw: string): string {
  const s = raw.trim();
  if (INJ_BECH32.test(s)) return s;
  if (ETH_HEX.test(s)) return getInjectiveAddress(s);
  throw new Error('Malformed address - expected inj1 (43 chars) or 0x (42 chars)');
}

app.post('/api/faucet', async (req, res) => {
  try {
    const receiver = normalizeAddress(req.body.address);
    const pk = PrivateKey.fromHex(process.env.FAUCET_PRIVATE_KEY!);
    const broadcaster = new MsgBroadcasterWithPk({
      network: Network.Testnet,
      privateKey: pk
    });

    const msg = MsgSend.fromJSON({
      amount: { denom: 'inj', amount: '1000000000000000000' }, // 1 INJ (18 decimals)
      srcInjectiveAddress: pk.toBech32(),
      dstInjectiveAddress: receiver
    });

    const result = await broadcaster.broadcast({ msgs: msg });
    res.json({ success: true, txHash: result.txHash });
  } catch (err: any) {
    res.status(400).json({ success: false, error: err.message });
  }
});
```

## Balance Verification Routines

### Comprehensive Multi-Wallet Balance Checker
```python
import asyncio
from pyinjective.async_client import AsyncClient
from pyinjective.core.network import Network

async def check_all_balances(wallets: list[dict], denom: str = "inj"):
    client = AsyncClient(Network.testnet())
    
    print(f"Checking balances for {len(wallets)} wallets...")
    for w in wallets:
        bank_res = await client.fetch_bank_balance(w["inj_address"], denom)
        balance = bank_res.get("balance", {}).get("amount", "0")
        
        sub_id = f"{w['eth_address'].lower()}000000000000000000000000"
        sub_res = await client.fetch_subaccount_deposits(sub_id)
        
        print(f"Wallet {w['inj_address'][:12]}... | Bank: {balance} | Subaccount: {sub_res}")

# Run verification
# asyncio.run(check_all_balances(wallets))
```

## Key Facts
- Max ~200 msgs per transaction (bounded by block gas limit).
- Use `MsgBroadcasterWithPk.new_using_simulation()` for dynamic gas estimation.
- Sequence mismatch errors (`account sequence mismatch`) can occur during high concurrency — implement exponential backoff retry.
- Funder wallet requires sufficient base denom (INJ) for both gas fees and outbound transfers.

## Dependencies

**Python (mass funding, batch MsgSend, subaccount deposits):**
- `injective-py>=1.12.0`
- `httpx>=0.27`
- Python >= 3.11

**Node (public faucet server + browser-origin address inputs):**
- `@injectivelabs/sdk-ts` — `PrivateKey`, `MsgSend`, `MsgBroadcasterWithPk`, `getInjectiveAddress`, `getEthereumAddress`
- `@injectivelabs/networks` — `Network.Testnet` / `Network.Mainnet`
- `express`, `dotenv`
- Node >= 20
