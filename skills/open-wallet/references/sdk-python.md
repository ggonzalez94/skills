# Python SDK Reference — Open Wallet Standard

Package: `open-wallet-standard` on PyPI. Native Rust core via PyO3 — no Rust toolchain required.

Prebuilt wheels: macOS (arm64, x64), Linux (x64, arm64). Python 3.9 - 3.13.

```python
from ows import (
    generate_mnemonic,
    derive_address,
    create_wallet,
    list_wallets,
    get_wallet,
    delete_wallet,
    rename_wallet,
    export_wallet,
    import_wallet_mnemonic,
    import_wallet_private_key,
    sign_message,
    sign_typed_data,
    sign_transaction,
    sign_and_send,
    create_policy,
    list_policies,
    get_policy,
    delete_policy,
    create_api_key,
    list_api_keys,
    revoke_api_key,
)
```

---

## Install

```bash
pip install open-wallet-standard
```

Verify:

```python
from ows import create_wallet
print(create_wallet("smoke-test"))
```

---

## Return Types

All functions return plain Python dicts (not dataclasses, not pydantic models).

### WalletInfo

```python
{
    "id": "a1b2c3d4-...",
    "name": "my-wallet",
    "created_at": "2026-01-15T09:30:00Z",
    "accounts": [
        {
            "chain_id": "eip155:1",
            "address": "0xAbC...",
            "derivation_path": "m/44'/60'/0'/0/0"
        },
        {
            "chain_id": "solana:5eykt4UsFv8P8NJdTREpY1vzqKqZKvdp",
            "address": "7xKX...",
            "derivation_path": "m/44'/501'/0'/0'"
        }
        # ... one entry per supported chain
    ]
}
```

### SignResult

```python
{
    "signature": "a1b2c3...ef",   # hex-encoded signature bytes
    "recovery_id": 1               # int for secp256k1 curves, None for Ed25519
}
```

### SendResult

```python
{
    "tx_hash": "0xabc123..."       # on-chain transaction hash
}
```

### ApiKeyResult

```python
{
    "token": "ows_key_a1b2c3d4...",   # shown ONCE — store it securely
    "id": "key-uuid-...",
    "name": "my-agent-key"
}
```

---

## Mnemonic Utilities

### generate_mnemonic

Generate a BIP-39 mnemonic phrase.

```python
from ows import generate_mnemonic

phrase_12 = generate_mnemonic()
phrase_24 = generate_mnemonic(words=24)
print(phrase_12)
# "abandon ability able about above absent absorb abstract absurd abuse access accident"
```

| Param | Type | Default | Description |
|-------|------|---------|-------------|
| `words` | `int` | `12` | Number of mnemonic words. Valid values: `12`, `15`, `18`, `21`, `24`. |

**Returns:** `str` — space-separated mnemonic phrase.

---

### derive_address

Derive a single address from a mnemonic for a given chain and account index.

```python
from ows import derive_address

mnemonic = "abandon ability able about above absent absorb abstract absurd abuse access accident"
evm_addr = derive_address(mnemonic, "evm")
sol_addr = derive_address(mnemonic, "solana", index=2)
btc_addr = derive_address(mnemonic, "bitcoin")

print(evm_addr)     # "0x..."
print(sol_addr)     # "7xKX..."
print(btc_addr)     # "bc1q..."
```

| Param | Type | Default | Description |
|-------|------|---------|-------------|
| `mnemonic` | `str` | *required* | BIP-39 mnemonic phrase. |
| `chain` | `str` | *required* | Chain family identifier. One of: `"evm"`, `"solana"`, `"sui"`, `"bitcoin"`, `"cosmos"`, `"tron"`, `"ton"`, `"spark"`, `"filecoin"`. |
| `index` | `int` | `0` | Account index in the derivation path. |

**Returns:** `str` — the derived address in the chain's native format.

---

## Wallet Management

### create_wallet

Create a new wallet. Generates a fresh BIP-39 mnemonic, derives addresses for all supported chains, and encrypts everything to disk.

```python
from ows import create_wallet

wallet = create_wallet("agent-treasury")
print(wallet["id"])          # "a1b2c3d4-..."
print(wallet["name"])        # "agent-treasury"
for acct in wallet["accounts"]:
    print(f"{acct['chain_id']}: {acct['address']}")
```

| Param | Type | Default | Description |
|-------|------|---------|-------------|
| `name` | `str` | *required* | Human-readable wallet name. Must be unique within the vault. |
| `passphrase` | `str` or `None` | `None` | Encryption passphrase. If `None`, the vault uses its default passphrase. |
| `words` | `int` | `12` | Mnemonic word count. Valid values: `12`, `15`, `18`, `21`, `24`. |
| `vault_path` | `str` or `None` | `None` | Custom vault directory. Defaults to `~/.ows/`. |

**Returns:** `dict` — WalletInfo.

---

### list_wallets

List all wallets in the vault.

```python
from ows import list_wallets

wallets = list_wallets()
for w in wallets:
    print(f"{w['name']} ({w['id']})")
```

| Param | Type | Default | Description |
|-------|------|---------|-------------|
| `vault_path` | `str` or `None` | `None` | Custom vault directory. Defaults to `~/.ows/`. |

**Returns:** `list[dict]` — list of WalletInfo dicts.

---

### get_wallet

Retrieve a single wallet by name or ID.

```python
from ows import get_wallet

wallet = get_wallet("agent-treasury")
# Also works with UUID:
wallet = get_wallet("a1b2c3d4-...")
```

| Param | Type | Default | Description |
|-------|------|---------|-------------|
| `name_or_id` | `str` | *required* | Wallet name or UUID. |
| `vault_path` | `str` or `None` | `None` | Custom vault directory. Defaults to `~/.ows/`. |

**Returns:** `dict` — WalletInfo.

---

### delete_wallet

Permanently delete a wallet from the vault. This is irreversible.

```python
from ows import delete_wallet

delete_wallet("old-wallet")
```

| Param | Type | Default | Description |
|-------|------|---------|-------------|
| `name_or_id` | `str` | *required* | Wallet name or UUID. |
| `vault_path` | `str` or `None` | `None` | Custom vault directory. Defaults to `~/.ows/`. |

**Returns:** `None`

---

### rename_wallet

Rename an existing wallet.

```python
from ows import rename_wallet

rename_wallet("agent-treasury", "production-treasury")
```

| Param | Type | Default | Description |
|-------|------|---------|-------------|
| `name_or_id` | `str` | *required* | Current wallet name or UUID. |
| `new_name` | `str` | *required* | New wallet name. Must be unique within the vault. |
| `vault_path` | `str` or `None` | `None` | Custom vault directory. Defaults to `~/.ows/`. |

**Returns:** `None`

---

### export_wallet

Export the wallet's mnemonic phrase. Handle the returned string with extreme care — never log or persist it.

```python
from ows import export_wallet

mnemonic = export_wallet("agent-treasury", passphrase="my-secret")
# mnemonic is a str of space-separated words — do NOT print or log this
```

| Param | Type | Default | Description |
|-------|------|---------|-------------|
| `name_or_id` | `str` | *required* | Wallet name or UUID. |
| `passphrase` | `str` or `None` | `None` | Decryption passphrase. Required if the wallet was created with one. |
| `vault_path` | `str` or `None` | `None` | Custom vault directory. Defaults to `~/.ows/`. |

**Returns:** `str` — BIP-39 mnemonic phrase.

---

## Import

### import_wallet_mnemonic

Import a wallet from an existing BIP-39 mnemonic. Derives addresses for all supported chains.

```python
from ows import import_wallet_mnemonic

wallet = import_wallet_mnemonic(
    "imported-wallet",
    "abandon ability able about above absent absorb abstract absurd abuse access accident"
)
print(wallet["accounts"][0]["address"])
```

| Param | Type | Default | Description |
|-------|------|---------|-------------|
| `name` | `str` | *required* | Name for the imported wallet. |
| `mnemonic` | `str` | *required* | BIP-39 mnemonic phrase (space-separated). |
| `passphrase` | `str` or `None` | `None` | Encryption passphrase for vault storage. |
| `index` | `int` or `None` | `None` | Account index override. If `None`, uses index 0. |
| `vault_path` | `str` or `None` | `None` | Custom vault directory. Defaults to `~/.ows/`. |

**Returns:** `dict` — WalletInfo with all chain accounts derived from the mnemonic.

---

### import_wallet_private_key

Import a wallet from a raw private key (hex-encoded). The chain parameter determines which cryptographic curve is used for the primary key. All 8 chain accounts are still generated.

```python
from ows import import_wallet_private_key

# Import a secp256k1 key (EVM, Bitcoin, Cosmos, Tron, Filecoin)
wallet = import_wallet_private_key(
    "evm-import",
    "4c0883a6917523...abcdef"
)

# Import an Ed25519 key (Solana, Sui, TON)
wallet = import_wallet_private_key(
    "sol-import",
    "3a5f8c...deadbeef",
    chain="solana"
)

# Provide both curve keys explicitly
wallet = import_wallet_private_key(
    "dual-import",
    "ignored",  # private_key_hex is ignored when explicit keys are given
    secp256k1_key="4c0883a6...",
    ed25519_key="3a5f8c..."
)
```

| Param | Type | Default | Description |
|-------|------|---------|-------------|
| `name` | `str` | *required* | Name for the imported wallet. |
| `private_key_hex` | `str` | *required* | Hex-encoded private key. Ignored when both `secp256k1_key` and `ed25519_key` are provided. |
| `chain` | `str` or `None` | `None` | Chain hint to determine curve. `"evm"` (default) uses secp256k1. `"solana"`, `"sui"`, `"ton"` use Ed25519. |
| `passphrase` | `str` or `None` | `None` | Encryption passphrase for vault storage. |
| `vault_path` | `str` or `None` | `None` | Custom vault directory. Defaults to `~/.ows/`. |
| `secp256k1_key` | `str` or `None` | `None` | Explicit secp256k1 private key (hex). |
| `ed25519_key` | `str` or `None` | `None` | Explicit Ed25519 private key (hex). |

**Curve selection logic:**

| `chain` value | Curve used |
|---------------|------------|
| `None` or `"evm"` | secp256k1 |
| `"bitcoin"` | secp256k1 |
| `"cosmos"` | secp256k1 |
| `"tron"` | secp256k1 |
| `"filecoin"` | secp256k1 |
| `"solana"` | Ed25519 |
| `"sui"` | Ed25519 |
| `"ton"` | Ed25519 |

When only one curve key is provided, OWS generates accounts for all 8 chains but only the chains matching the provided curve will have valid addresses. When both `secp256k1_key` and `ed25519_key` are provided, all chain accounts are fully valid.

**Returns:** `dict` — WalletInfo with all 8 chain accounts.

---

## Signing

### sign_message

Sign an arbitrary message with the wallet's private key for the specified chain.

```python
from ows import sign_message

# EVM — produces EIP-191 personal_sign
sig = sign_message("agent-treasury", "evm", "hello world")
print(sig["signature"])     # "a1b2c3...ef"
print(sig["recovery_id"])   # 1

# Solana — Ed25519 signature
sig = sign_message("agent-treasury", "solana", "hello world")
print(sig["recovery_id"])   # None (Ed25519 has no recovery ID)

# With passphrase or API key token
sig = sign_message(
    "agent-treasury", "evm", "hello world",
    passphrase="ows_key_a1b2c3d4..."
)

# Custom encoding
sig = sign_message(
    "agent-treasury", "evm", "68656c6c6f",
    encoding="hex"
)

# Non-default account index
sig = sign_message(
    "agent-treasury", "evm", "hello",
    index=2
)
```

| Param | Type | Default | Description |
|-------|------|---------|-------------|
| `wallet` | `str` | *required* | Wallet name or UUID. |
| `chain` | `str` | *required* | Chain family: `"evm"`, `"solana"`, `"sui"`, `"bitcoin"`, `"cosmos"`, `"tron"`, `"ton"`, `"spark"`, `"filecoin"`. |
| `message` | `str` | *required* | Message to sign. Interpreted as UTF-8 by default. |
| `passphrase` | `str` or `None` | `None` | Owner passphrase or `ows_key_...` API token. If an API token is provided, policies are evaluated before signing. |
| `encoding` | `str` or `None` | `None` | Message encoding. `None` for UTF-8 string, `"hex"` for hex-encoded bytes. |
| `index` | `int` or `None` | `None` | Account index. Defaults to 0. |
| `vault_path` | `str` or `None` | `None` | Custom vault directory. Defaults to `~/.ows/`. |

**Returns:** `dict` — SignResult.

---

### sign_typed_data

Sign EIP-712 typed structured data. EVM only.

```python
import json
from ows import sign_typed_data

typed_data = {
    "types": {
        "EIP712Domain": [
            {"name": "name", "type": "string"},
            {"name": "version", "type": "string"},
            {"name": "chainId", "type": "uint256"},
            {"name": "verifyingContract", "type": "address"}
        ],
        "Transfer": [
            {"name": "to", "type": "address"},
            {"name": "amount", "type": "uint256"}
        ]
    },
    "primaryType": "Transfer",
    "domain": {
        "name": "MyDApp",
        "version": "1",
        "chainId": 8453,
        "verifyingContract": "0x1234567890abcdef1234567890abcdef12345678"
    },
    "message": {
        "to": "0xabcdefabcdefabcdefabcdefabcdefabcdefabcd",
        "amount": 1000000
    }
}

sig = sign_typed_data(
    "agent-treasury",
    "evm",
    json.dumps(typed_data)
)
print(sig["signature"])
print(sig["recovery_id"])
```

| Param | Type | Default | Description |
|-------|------|---------|-------------|
| `wallet` | `str` | *required* | Wallet name or UUID. |
| `chain` | `str` | *required* | Must be `"evm"` (EIP-712 is EVM-only). |
| `typed_data_json` | `str` | *required* | JSON string of the EIP-712 typed data structure. |
| `passphrase` | `str` or `None` | `None` | Owner passphrase or `ows_key_...` API token. |
| `index` | `int` or `None` | `None` | Account index. Defaults to 0. |
| `vault_path` | `str` or `None` | `None` | Custom vault directory. Defaults to `~/.ows/`. |

**Returns:** `dict` — SignResult.

---

### sign_transaction

Sign a raw transaction without broadcasting. Returns the signature that can be attached and submitted separately.

```python
from ows import sign_transaction

# EVM transaction (RLP-encoded, hex)
sig = sign_transaction(
    "agent-treasury",
    "evm",
    "02f87083014a3401843b9aca00850d09dc300082520894..."
)
print(sig["signature"])

# Using an API key token (policy-gated)
sig = sign_transaction(
    "agent-treasury",
    "base",
    "02f870...",
    passphrase="ows_key_a1b2c3d4..."
)
```

| Param | Type | Default | Description |
|-------|------|---------|-------------|
| `wallet` | `str` | *required* | Wallet name or UUID. |
| `chain` | `str` | *required* | Chain family or specific chain shorthand (e.g., `"evm"`, `"base"`, `"solana"`). |
| `tx_hex` | `str` | *required* | Hex-encoded raw transaction bytes. |
| `passphrase` | `str` or `None` | `None` | Owner passphrase or `ows_key_...` API token. |
| `index` | `int` or `None` | `None` | Account index. Defaults to 0. |
| `vault_path` | `str` or `None` | `None` | Custom vault directory. Defaults to `~/.ows/`. |

**Returns:** `dict` — SignResult.

---

### sign_and_send

Sign a transaction and broadcast it to the network. Returns the on-chain transaction hash.

```python
from ows import sign_and_send

result = sign_and_send(
    "agent-treasury",
    "base",
    "02f870..."
)
print(result["tx_hash"])  # "0xabc123..."

# With a custom RPC endpoint
result = sign_and_send(
    "agent-treasury",
    "base",
    "02f870...",
    rpc_url="https://mainnet.base.org"
)

# Policy-gated via API key
result = sign_and_send(
    "agent-treasury",
    "base",
    "02f870...",
    passphrase="ows_key_a1b2c3d4..."
)
```

| Param | Type | Default | Description |
|-------|------|---------|-------------|
| `wallet` | `str` | *required* | Wallet name or UUID. |
| `chain` | `str` | *required* | Chain family or specific chain shorthand. |
| `tx_hex` | `str` | *required* | Hex-encoded raw transaction bytes. |
| `passphrase` | `str` or `None` | `None` | Owner passphrase or `ows_key_...` API token. |
| `index` | `int` or `None` | `None` | Account index. Defaults to 0. |
| `rpc_url` | `str` or `None` | `None` | Custom RPC endpoint URL. If `None`, OWS uses its default RPC for the chain. |
| `vault_path` | `str` or `None` | `None` | Custom vault directory. Defaults to `~/.ows/`. |

**Returns:** `dict` — SendResult.

---

## Policy Management

Policies constrain what operations an API key can perform. They are evaluated before every signing operation. All policies attached to an API key are AND-combined (every policy must allow).

### create_policy

Create a policy from a JSON definition.

```python
import json
from ows import create_policy

policy = {
    "id": "base-only",
    "name": "Restrict to Base chain",
    "version": 1,
    "created_at": "2026-01-01T00:00:00Z",
    "rules": [
        {"type": "allowed_chains", "chain_ids": ["eip155:8453"]},
        {"type": "expires_at", "timestamp": "2026-12-31T23:59:59Z"}
    ],
    "action": "deny"
}

create_policy(json.dumps(policy))
```

| Param | Type | Default | Description |
|-------|------|---------|-------------|
| `policy_json` | `str` | *required* | JSON string of the policy definition. Must include `id`, `name`, `version`, `created_at`, `rules`, and `action` fields. |
| `vault_path` | `str` or `None` | `None` | Custom vault directory. Defaults to `~/.ows/`. |

**Returns:** `None`

**Policy rule types:**

| Rule type | Fields | Description |
|-----------|--------|-------------|
| `allowed_chains` | `chain_ids: list[str]` | Only allow signing for the listed CAIP-2 chain IDs. |
| `expires_at` | `timestamp: str` | Deny all operations after this ISO-8601 timestamp. |

---

### list_policies

List all policies in the vault.

```python
from ows import list_policies

policies = list_policies()
for p in policies:
    print(f"{p['id']}: {p['name']}")
```

| Param | Type | Default | Description |
|-------|------|---------|-------------|
| `vault_path` | `str` or `None` | `None` | Custom vault directory. Defaults to `~/.ows/`. |

**Returns:** `list[dict]` — list of policy dicts.

---

### get_policy

Retrieve a single policy by ID.

```python
from ows import get_policy

policy = get_policy("base-only")
print(policy["rules"])
```

| Param | Type | Default | Description |
|-------|------|---------|-------------|
| `id` | `str` | *required* | Policy ID. |
| `vault_path` | `str` or `None` | `None` | Custom vault directory. Defaults to `~/.ows/`. |

**Returns:** `dict` — policy definition.

---

### delete_policy

Delete a policy from the vault.

```python
from ows import delete_policy

delete_policy("base-only")
```

| Param | Type | Default | Description |
|-------|------|---------|-------------|
| `id` | `str` | *required* | Policy ID. |
| `vault_path` | `str` or `None` | `None` | Custom vault directory. Defaults to `~/.ows/`. |

**Returns:** `None`

---

## API Key Management

API keys give agents scoped access to wallets. The agent uses the token in place of a passphrase. OWS detects the `ows_key_` prefix and evaluates attached policies before every signing operation.

### create_api_key

Create an API key bound to specific wallets and policies.

```python
from ows import create_api_key

result = create_api_key(
    name="deploy-bot",
    wallet_ids=["agent-treasury"],
    policy_ids=["base-only"],
    passphrase="owner-passphrase"
)
print(result["token"])  # "ows_key_a1b2c3d4..." — save this, shown ONCE
print(result["id"])     # "key-uuid-..."
print(result["name"])   # "deploy-bot"

# With expiration
result = create_api_key(
    name="temp-bot",
    wallet_ids=["agent-treasury"],
    policy_ids=["base-only"],
    passphrase="owner-passphrase",
    expires_at="2026-06-30T23:59:59Z"
)
```

| Param | Type | Default | Description |
|-------|------|---------|-------------|
| `name` | `str` | *required* | Human-readable name for the key. |
| `wallet_ids` | `list[str]` | *required* | List of wallet names or UUIDs the key can access. |
| `policy_ids` | `list[str]` | *required* | List of policy IDs to attach. All are AND-combined. |
| `passphrase` | `str` | *required* | Owner passphrase to authorize key creation. |
| `expires_at` | `str` or `None` | `None` | ISO-8601 expiration timestamp. `None` means no key-level expiration (policies may still enforce time limits). |
| `vault_path` | `str` or `None` | `None` | Custom vault directory. Defaults to `~/.ows/`. |

**Returns:** `dict` — ApiKeyResult. The `token` field is only shown at creation time.

---

### list_api_keys

List all API keys in the vault (tokens are NOT included — only metadata).

```python
from ows import list_api_keys

keys = list_api_keys()
for k in keys:
    print(f"{k['name']} (id={k['id']})")
```

| Param | Type | Default | Description |
|-------|------|---------|-------------|
| `vault_path` | `str` or `None` | `None` | Custom vault directory. Defaults to `~/.ows/`. |

**Returns:** `list[dict]` — list of API key metadata dicts.

---

### revoke_api_key

Revoke an API key immediately. The token becomes invalid and the associated encrypted mnemonic copy is deleted.

```python
from ows import revoke_api_key

revoke_api_key("key-uuid-...")
```

| Param | Type | Default | Description |
|-------|------|---------|-------------|
| `id` | `str` | *required* | API key UUID (from `create_api_key` or `list_api_keys`). |
| `vault_path` | `str` or `None` | `None` | Custom vault directory. Defaults to `~/.ows/`. |

**Returns:** `None`

---

## Custom Vault Path

Every function accepts an optional `vault_path` parameter that overrides the default `~/.ows/` directory. This is essential for testing isolation — each test run can use its own vault so wallets, policies, and keys never collide.

```python
import tempfile
from ows import create_wallet, list_wallets, create_policy, create_api_key, sign_message
import json

# Create an isolated vault for testing
vault = tempfile.mkdtemp(prefix="ows-test-")

# All operations target this isolated vault
wallet = create_wallet("test-wallet", vault_path=vault)
print(wallet["name"])  # "test-wallet"

# List wallets — only sees wallets in this vault
wallets = list_wallets(vault_path=vault)
assert len(wallets) == 1

# Create a policy in the isolated vault
policy = {
    "id": "test-policy",
    "name": "Test policy",
    "version": 1,
    "created_at": "2026-01-01T00:00:00Z",
    "rules": [
        {"type": "allowed_chains", "chain_ids": ["eip155:8453"]}
    ],
    "action": "deny"
}
create_policy(json.dumps(policy), vault_path=vault)

# Create an API key in the isolated vault
key = create_api_key(
    name="test-key",
    wallet_ids=["test-wallet"],
    policy_ids=["test-policy"],
    passphrase="test-pass",
    vault_path=vault
)

# Sign using the API key — policies are enforced
sig = sign_message(
    "test-wallet", "evm", "hello from test",
    passphrase=key["token"],
    vault_path=vault
)
print(sig["signature"])

# Cleanup: just delete the temp directory
import shutil
shutil.rmtree(vault)
```

Use this pattern in pytest fixtures:

```python
import pytest
import tempfile
import shutil
from ows import create_wallet

@pytest.fixture
def vault(tmp_path):
    """Provide an isolated OWS vault for each test."""
    vault_dir = str(tmp_path / "ows-vault")
    yield vault_dir
    # tmp_path is cleaned up automatically by pytest

def test_create_wallet(vault):
    wallet = create_wallet("my-wallet", vault_path=vault)
    assert wallet["name"] == "my-wallet"
    assert len(wallet["accounts"]) > 0

def test_wallets_are_isolated(vault):
    """Each test gets its own empty vault."""
    from ows import list_wallets
    assert list_wallets(vault_path=vault) == []
```

---

## Complete End-to-End Example

A full workflow: create a wallet, set up policy-gated agent access, sign a transaction, and clean up.

```python
import json
import tempfile
import shutil
from ows import (
    create_wallet,
    get_wallet,
    create_policy,
    create_api_key,
    sign_message,
    sign_transaction,
    sign_and_send,
    list_wallets,
    revoke_api_key,
    delete_wallet,
)

vault = tempfile.mkdtemp(prefix="ows-demo-")

# --- Step 1: Create a wallet ---
wallet = create_wallet("demo-treasury", vault_path=vault)
evm_account = next(a for a in wallet["accounts"] if a["chain_id"] == "eip155:1")
print(f"EVM address: {evm_account['address']}")

# --- Step 2: Define a policy ---
policy = {
    "id": "base-usdc-only",
    "name": "Base chain, expires end of year",
    "version": 1,
    "created_at": "2026-01-01T00:00:00Z",
    "rules": [
        {"type": "allowed_chains", "chain_ids": ["eip155:8453"]},
        {"type": "expires_at", "timestamp": "2026-12-31T23:59:59Z"}
    ],
    "action": "deny"
}
create_policy(json.dumps(policy), vault_path=vault)

# --- Step 3: Create a scoped API key ---
key = create_api_key(
    name="demo-agent",
    wallet_ids=[wallet["id"]],
    policy_ids=["base-usdc-only"],
    passphrase="owner-secret",
    vault_path=vault
)
agent_token = key["token"]  # This is what the agent receives

# --- Step 4: Agent signs using the token ---
# The agent never sees the private key. OWS evaluates the
# "base-usdc-only" policy before allowing the signature.
sig = sign_message(
    "demo-treasury",
    "base",
    "Approve transfer of 100 USDC",
    passphrase=agent_token,
    vault_path=vault
)
print(f"Signature: {sig['signature'][:20]}...")

# --- Step 5: Revoke access when done ---
revoke_api_key(key["id"], vault_path=vault)

# --- Cleanup ---
delete_wallet("demo-treasury", vault_path=vault)
shutil.rmtree(vault)
```

---

## Supported Chains

Every wallet automatically derives addresses for all chains below from a single mnemonic.

| Chain | `chain` param | Curve | CAIP-2 ID (mainnet) |
|-------|---------------|-------|----------------------|
| EVM (Ethereum, Base, etc.) | `"evm"` | secp256k1 | `eip155:1`, `eip155:8453`, ... |
| Solana | `"solana"` | Ed25519 | `solana:5eykt4UsFv8P8NJdTREpY1vzqKqZKvdp` |
| Bitcoin | `"bitcoin"` | secp256k1 | `bip122:000000000019d6689c085ae165831e93` |
| Cosmos | `"cosmos"` | secp256k1 | `cosmos:cosmoshub-4` |
| Tron | `"tron"` | secp256k1 | `tron:mainnet` |
| TON | `"ton"` | Ed25519 | `ton:mainnet` |
| Sui | `"sui"` | Ed25519 | `sui:mainnet` |
| Filecoin | `"filecoin"` | secp256k1 | `fil:mainnet` |
| Spark | `"spark"` | secp256k1 | `spark:mainnet` |

---

## Error Handling

All functions raise Python exceptions on failure. Common patterns:

```python
from ows import get_wallet, sign_message

# Wallet not found
try:
    get_wallet("nonexistent")
except Exception as e:
    print(f"Error: {e}")  # wallet not found

# Policy denial (when using API key token)
try:
    sig = sign_message(
        "my-wallet", "solana", "hello",
        passphrase="ows_key_abc123..."
    )
except Exception as e:
    print(f"Policy denied: {e}")  # e.g., chain not in allowed_chains
```

There is no custom exception hierarchy — all errors are raised as standard Python `Exception` (or subclasses). Check the message string for specifics.

---

## Security Reminders

1. **Never log or print mnemonics or private keys.** The return value of `export_wallet` is the raw mnemonic. Handle it only in memory.
2. **Use API key tokens for agent access.** Pass `ows_key_...` tokens as the `passphrase` parameter. The owner passphrase bypasses all policies.
3. **Scope API keys with policies.** At minimum, use `allowed_chains` and `expires_at`.
4. **Revoke tokens when no longer needed.** `revoke_api_key` is instant and permanent.
5. **Use `vault_path` for test isolation.** Never let tests touch `~/.ows/`.
