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

Fund many Injective wallets efficiently. Batch bank transfers, deposit to exchange subaccounts, and verify balances. Based on an Injective RFQ test framework.

## Core Funding Routines

The routines below provide self-contained Python (`pyinjective`) and TypeScript (`@injectivelabs/sdk-ts`) patterns for mass wallet funding, multi-denom bank transfers, exchange subaccount deposits, and automated balance checks without requiring external repository scripts.

## Batch Bank Transfers

### 200 Transfers in One Transaction
Batch up to 200 `MsgSend` in a single transaction. **Enforce the limit** before broadcasting — the Injective node rejects transactions that exceed the gas cap for large batches:
```python
from pyinjective.composer import Composer
from pyinjective.transaction import Transaction

MAX_MSGS_PER_TX = 200  # hard cap enforced by node gas limit

composer = Composer(network=network.string())
msgs = []

if len(wallets) > MAX_MSGS_PER_TX:
    raise ValueError(
        f"Batch too large: {len(wallets)} wallets exceeds the {MAX_MSGS_PER_TX}-message limit. "
        "Split into smaller chunks and broadcast separately."
    )

for wallet in wallets:
    # Send INJ
    msgs.append(composer.msg_send(
        sender=funder_address,
        receiver=wallet.inj_address,
        amount=inj_amount,  # in wei (1e18)
        denom="inj"
    ))
    # Send USDT
    msgs.append(composer.msg_send(
        sender=funder_address,
        receiver=wallet.inj_address,
        amount=usdt_amount,  # in 1e6
        denom=usdt_denom
    ))

# Build and broadcast single tx with all msgs
tx = Transaction(msgs=msgs, ...)
```

To process more than 200 wallets, split into chunks:
```python
import math

CHUNK_SIZE = 100  # msgs per wallet × chunk = total msgs per tx
for i in range(0, len(wallets), CHUNK_SIZE):
    chunk = wallets[i:i + CHUNK_SIZE]
    # build and broadcast each chunk independently
```

### Quote-asset denoms by network
| Network | Asset | Denom | Decimals |
|---------|-------|-------|----------|
| Mainnet | USDT  | `peggy0xdAC17F958D2ee523a2206206994597C13D831ec7` | 6 |
| Mainnet | USDC  | `peggy0xA0b86991c6218b36c1d19D4a2e9Eb0cE3606eB48` | 6 |
| Testnet | USDT  | `peggy0x87aB3B4C8661e07D6372361211B96ed4Dc36B1B5` | 6 |
| Testnet | USDC  | `erc20:0x0C382e685bbeeFE5d3d9C29e29E341fEE8E84C5d` | 6 |
| Devnet  | varies per config | — | 6 |

Which asset a given RFQ/exchange market uses depends on its `quote_denom`. Query the market via LCD (`/injective/exchange/v2/derivative/markets/{id}`) and look at `market.quote_denom` — never hardcode.

### INJ Amounts
INJ uses 18 decimals (like ETH wei):
- 1 INJ = `1000000000000000000` (1e18)
- 0.1 INJ = `100000000000000000` (1e17)

## Subaccount Deposits

After funding bank accounts, move to exchange subaccounts for trading:

```python
from pyinjective.core.broadcaster import MsgBroadcasterWithPk

msg = composer.msg_subaccount_deposit(
    sender=address,
    subaccount_id=get_subaccount_id(address, nonce=0),
    amount=Decimal(str(amount)),
    denom=denom
)

broadcaster = MsgBroadcasterWithPk.new_using_simulation(
    network=network,
    private_key=private_key
)
await broadcaster.broadcast([msg])
```

### Subaccount ID Calculation

Canonicalize and validate at every construction site — strip `0x` prefix, require exactly 40 hex characters for the address part, and left-pad the nonce to 24 hex characters:

```python
import re

ETH_ADDR_RE = re.compile(r'^(?:0x)?([0-9a-fA-F]{40})$')

def get_subaccount_id(inj_address: str, nonce: int = 0) -> str:
    """Generate canonical subaccount ID: eth_address (lowercase, no 0x) + nonce (24 hex chars).
    
    All three construction sites (bank, subaccount deposit, and authz lookups) must use this
    helper — never inline the concatenation — so that IDs are always in canonical form.
    """
    eth_raw = inj_to_eth_address(inj_address)  # returns '0x...' form
    m = ETH_ADDR_RE.match(eth_raw)
    if not m:
        raise ValueError(f"Invalid ETH address derived from {inj_address!r}: {eth_raw!r}")
    eth_hex = m.group(1).lower()               # strip 0x, lowercase, exactly 40 chars
    return eth_hex + format(nonce, '024x')     # 40 + 24 = 64 hex chars total
```

## Two-Step USDT Funding (Testnet Pattern)

1. **Parent sends USDT to child** via bank `MsgSend`
2. **Child deposits USDT** from bank to exchange subaccount

This is needed because on testnet, the public faucet only gives INJ, not USDT/USDC. You need to pre-fund a parent wallet with the quote asset.

## Public Faucet Server (Accept-Either-Address Pattern)

Pattern for a browser-facing faucet that takes a caller-supplied address and sends INJ + quote-asset in a single `MsgSend`. Proven working against testnet INJ/USDC PERP (2026-04-20; tx `EDE9788844E1780DC72ABBC67346550A85E842A57BF9A451FDDD130125B98163`).

### Principles

- **Accept both `inj1…` and `0x…`** — same secp256k1 key, two encodings. MetaMask/bridge users only have the `0x` form; Cosmos-native users paste `inj1`. Forcing one side creates a support tax. See **injective-wallet-ops** for the conversion helpers.
- **Canonicalize to `inj1` at the boundary.** `MsgSend` wants bech32 receivers; more importantly, rate limits + dedup keys must be keyed on the canonical form. Otherwise a caller can flip encodings to dodge a cooldown.
- **Per-request hard caps** (e.g. 500 USDC + 2 INJ) enforced server-side, not trusted from the request body.
- **Per-address cooldown** in a file-backed map (`{ addr: { lastRequestAt, lastTxHash } }`). SQLite is overkill for testnet volumes.
- **Status endpoint** that returns hot-wallet balance + remaining budget. Useful for dashboards and for `npm run dev`-time debugging; optional to expose publicly.
- **Fresh hot wallet per deployment**, not reused across envs. Fund it with just enough for ~100 requests so a leak is a bounded loss. Top up on demand.

### Reference implementation (Node + `@injectivelabs/sdk-ts`)

```javascript
import 'dotenv/config';
import express from 'express';
import {
  PrivateKey, MsgSend, MsgBroadcasterWithPk,
  getInjectiveAddress, getEthereumAddress,
} from '@injectivelabs/sdk-ts';
import { Network } from '@injectivelabs/networks';

const FAUCET_PK  = process.env.FAUCET_PRIVATE_KEY;
const USDC_DENOM = 'erc20:0x0C382e685bbeeFE5d3d9C29e29E341fEE8E84C5d';  // testnet
const USDC_SUB   = (500n  * 10n**6n ).toString();   // 500 USDC
const INJ_SUB    = (2n    * 10n**18n).toString();   //   2 INJ
const COOLDOWN_MS = 24 * 60 * 60 * 1000;

const priv         = PrivateKey.fromHex('0x' + FAUCET_PK.replace(/^0x/, ''));
const FAUCET_ADDR  = priv.toBech32();
const broadcaster  = new MsgBroadcasterWithPk({ privateKey: FAUCET_PK, network: Network.Testnet });

// 1. Accept either form, canonicalize to inj1 for downstream use
const INJ_BECH32 = /^inj1[02-9ac-hj-np-z]{38}$/;
const ETH_HEX    = /^0x[0-9a-fA-F]{40}$/;
function normalize(raw) {
  const s = (raw || '').trim();
  if (INJ_BECH32.test(s)) return { inj: s,                      eth: getEthereumAddress(s).toLowerCase() };
  if (ETH_HEX.test(s))    return { inj: getInjectiveAddress(s), eth: s.toLowerCase() };
  throw new Error('malformed address — expected inj1… (43 chars) or 0x… (42 chars)');
}

// 2. File-backed rate limit, keyed on the inj form
const limits = new Map();  // load from disk on boot, persist on each record

// 3. The send itself — a single MsgSend with both denoms in one tx
async function sendFunds(recipientInj) {
  const msg = MsgSend.fromJSON({
    srcInjectiveAddress: FAUCET_ADDR,
    dstInjectiveAddress: recipientInj,
    amount: [{ denom: USDC_DENOM, amount: USDC_SUB }, { denom: 'inj', amount: INJ_SUB }],
  });
  const tx = await broadcaster.broadcast({ msgs: [msg] });
  return tx.txHash;
}

const app = express();
app.use(express.json());
app.post('/api/faucet', async (req, res) => {
  let parsed;
  try { parsed = normalize(req.body?.address); }
  catch (e) { return res.status(400).json({ error: e.message }); }

  const prior = limits.get(parsed.inj);
  if (prior && Date.now() - prior.lastRequestAt < COOLDOWN_MS) {
    return res.status(429).json({ error: 'cooldown', retry_after_seconds: /* … */ 0 });
  }
  try {
    const txHash = await sendFunds(parsed.inj);
    limits.set(parsed.inj, { lastRequestAt: Date.now(), lastTxHash: txHash });
    // persist limits to disk
    res.json({ ok: true, tx_hash: txHash, recipient: parsed });
  } catch (e) {
    res.status(500).json({ error: e.message || 'send failed' });
  }
});
app.listen(46001);
```

### Gotchas

- **First send to an empty account** may log "account not found" before the tx completes — normal, the first `MsgSend` creates the bank account on-chain.
- **INJ gas is the tighter budget** than USDC. A 50k-USDC / 100-INJ seed gives ~50 requests (100 INJ / 2 per req) before the INJ side drains even though USDC is nowhere near empty. Monitor both.
- Expose the status endpoint behind the same basic-auth / cookie gate as the rest of your site if you don't want randos hitting it from the open internet.
- Put the faucet behind a dedicated subdomain (e.g. `faucet.example.xyz`) or a `/api/faucet` nginx `location` block on an existing domain — a separate pm2 process, not inside `pm2 serve` (which is static-only).

## Balance Checks

### Bank Balance
```python
balance = await client.fetch_bank_balance(address, denom)
# Returns {"balance": {"denom": "...", "amount": "..."}}
```

### Subaccount Balance
```python
deposits = await client.fetch_subaccount_deposits(subaccount_id)
# Returns available + total deposits per denom
```

### Funding Verification Routine
```python
# Check if all wallets are properly funded
async def verify_funding(wallets: list[str], client, denom: str = "inj"):
    for address in wallets:
        bank = await client.fetch_bank_balance(address, denom)
        sub_id = get_subaccount_id(address, nonce=0)
        deposits = await client.fetch_subaccount_deposits(sub_id)
        print(f"Wallet: {address[:12]}... | Bank: {bank} | Subaccount: {deposits}")
```

## Common Execution Patterns

- **Batch Send**: Group child addresses into chunks of up to 200 `MsgSend` messages per transaction.
- **Two-Step Quote Asset Funding**: Parent wallet bank-transfers USDT/USDC to child wallets; each child wallet then invokes `msg_subaccount_deposit` to move the quote asset into subaccount `0`.
- **Pre-flight Gas Checks**: Ensure the funder hot wallet holds sufficient native INJ for simulation and execution gas before broadcasting multi-message batches.

## Key Facts
- Max ~200 msgs per transaction (gas limit)
- Use `MsgBroadcasterWithPk.new_using_simulation()` for auto gas estimation
- Sequence mismatch errors are common in rapid-fire txs -- add retry logic
- Always verify wallet balances using the verification routine after funding
- Funder wallet needs enough INJ for gas + enough tokens for all transfers

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
