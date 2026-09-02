---
name: injective-wallet-ops
description: Mass create, derive, and manage Injective wallets. Generate wallets from mnemonics (BIP-44 HD derivation), create random wallets, convert between ETH/INJ addresses, and batch fund wallets with INJ or USDT. Supports bulk wallet generation and batch funding.
license: MIT
metadata:
  author: ck
  version: "1.0.0"
---

# Injective Wallet Operations Skill

## Overview

Generate, derive, fund, and manage Injective wallets in bulk using self-contained Python and TypeScript routines.

## Wallet Generation

### From Mnemonic (Deterministic, BIP-44)
Derive any number of Injective-compatible wallets deterministically from a 12/24 word BIP-39 mnemonic seed phrase:
```python
import bech32
from eth_account import Account

Account.enable_unaudited_hdwallet_features()

def eth_to_inj_address(eth_address: str) -> str:
    eth_bytes = bytes.fromhex(eth_address.removeprefix("0x"))
    five_bit = bech32.convertbits(eth_bytes, 8, 5)
    return bech32.bech32_encode("inj", five_bit)

def generate_wallets_from_seed(seed_phrase: str, count: int, start_index: int = 0) -> list[dict]:
    wallets = []
    for i in range(start_index, start_index + count):
        path = f"m/44'/60'/0'/0/{i}"
        acct = Account.from_mnemonic(seed_phrase, account_path=path)
        eth_addr = acct.address
        inj_addr = eth_to_inj_address(eth_addr)
        wallets.append({
            "index": i,
            "private_key": acct.key.hex(),
            "eth_address": eth_addr,
            "inj_address": inj_addr,
        })
    return wallets
```

### Random Keypair Generation
```python
import secrets
from pyinjective.core.account import PrivateKey

def create_random_wallet() -> dict:
    raw_hex = secrets.token_hex(32)
    pk = PrivateKey.from_hex(raw_hex)
    return {
        "private_key": raw_hex,
        "inj_address": pk.to_address().to_acc_bech32(),
        "eth_address": pk.to_address().to_hex(),
    }
```

### Address Conversion (ETH <-> INJ)

Injective is EVM-compatible: every secp256k1 key has two address encodings that map 1:1. The `0x` form (40 hex chars) is what MetaMask + bridging tools show; the `inj1` form (bech32, 38 chars after the prefix) is what Cosmos RPCs, `MsgSend`, contract state, and the indexer use. Any code that accepts a user-supplied address should accept both forms and canonicalize to `inj1` before chain ops — **rate limits, dedup keys, and authz lookups must be keyed on the inj form**, otherwise a caller can dodge them by flipping encodings.

Validation regexes (accept both, reject everything else):
```
inj1:  ^inj1[02-9ac-hj-np-z]{38}$      (bech32 charset; no b/i/o/1)
0x:    ^0x[0-9a-fA-F]{40}$              (mixed case OK; normalize to lowercase)
```

**Python** (standalone, no SDK):
```python
import bech32

def eth_to_inj_address(eth_address: str) -> str:
    eth_bytes = bytes.fromhex(eth_address.removeprefix("0x"))
    five_bit = bech32.convertbits(eth_bytes, 8, 5)
    return bech32.bech32_encode("inj", five_bit)

def inj_to_eth_address(inj_address: str) -> str:
    _, five_bit = bech32.bech32_decode(inj_address)
    eight_bit = bech32.convertbits(five_bit, 5, 8, False)
    return "0x" + bytes(eight_bit).hex()
```

**Node / TypeScript** (via `@injectivelabs/sdk-ts`):
```ts
import { getInjectiveAddress, getEthereumAddress } from '@injectivelabs/sdk-ts';

// 0x -> inj1
const inj = getInjectiveAddress('0xYourEthAddress...40hex');
// inj1 -> 0x (lowercase)
const eth = getEthereumAddress('inj1yourbech32address...');
```

**Accept-either pattern** (server accepting an external address):
```ts
const INJ_BECH32 = /^inj1[02-9ac-hj-np-z]{38}$/;
const ETH_HEX    = /^0x[0-9a-fA-F]{40}$/;

function normalize(raw: string) {
  const s = raw.trim();
  if (INJ_BECH32.test(s)) return { inj: s, eth: getEthereumAddress(s).toLowerCase() };
  if (ETH_HEX.test(s))    return { inj: getInjectiveAddress(s), eth: s.toLowerCase() };
  throw new Error('malformed address - expected inj1 (43 chars) or 0x (42 chars)');
}
```

## Mass Funding & Subaccounts

### Send INJ to Many Wallets (Batched MsgSend)
Batch up to 200 `MsgSend` messages in a single transaction:
```python
from decimal import Decimal
from pyinjective.composer import Composer

msgs = []
for wallet in wallets:
    msg = composer.msg_send(
        sender=funder_address,
        receiver=wallet["inj_address"],
        amount=Decimal("1.0"),
        denom="inj"
    )
    msgs.append(msg)
```

### Subaccount ID Construction
Construct the default subaccount (nonce = 0) for any Injective address:
```python
def get_default_subaccount_id(eth_address: str) -> str:
    return f"{eth_address.lower().removeprefix('0x').rjust(40, '0')}000000000000000000000000"
```

## Balance Checks
```python
# Bank balance lookup
balance = await client.fetch_bank_balance(address, denom)

# Subaccount deposit lookup
deposits = await client.fetch_subaccount_deposits(subaccount_id)
```

## Dependencies
- `injective-py>=1.12.0`
- `eth-account>=0.11.0`
- `bech32>=1.2.0`
- Python >= 3.11
