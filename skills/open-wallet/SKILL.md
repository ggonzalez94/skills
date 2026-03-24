---
name: open-wallet
description: Open Wallet Standard (OWS) — the universal wallet layer for AI agents. Use this skill whenever the task involves creating crypto wallets, signing transactions or messages, managing wallet policies, setting up agent access with API keys, importing/exporting wallets, checking balances, paying for services (x402), or any on-chain operation that needs a local wallet. Triggers on OWS, open wallet, wallet create, wallet sign, wallet policy, API key for wallet, CAIP chain IDs, multi-chain wallet, agent wallet access, ows CLI, @open-wallet-standard, or any mention of local-first wallet management for agents.
---

# Open Wallet Standard (OWS)

OWS is an open standard for local wallet storage, delegated agent access, and policy-gated signing. One wallet, one encrypted vault on disk (`~/.ows/`), works across every chain — EVM, Solana, Bitcoin, Cosmos, Tron, TON, Sui, Filecoin, and more. Agents get scoped API tokens (never raw keys) with policy enforcement before any signing happens.

**Why it matters for agents**: Before OWS, every agent framework rolled its own key management. OWS gives you a single wallet that any tool can use — CLI, Node.js SDK, Python SDK, MCP server — with the same security model everywhere.

---

## Install

Pick your interface:

```bash
# Everything (CLI + Node.js + Python bindings)
curl -fsSL https://docs.openwallet.sh/install.sh | bash

# Node.js only
npm install @open-wallet-standard/core

# Python only
pip install open-wallet-standard
```

The CLI binary lives at `~/.ows/bin/ows`. The Node.js and Python packages include prebuilt native binaries (Rust core via NAPI/PyO3) — no Rust toolchain needed.

---

## Core Workflows

### 1. Create a Wallet

A single command derives addresses for every supported chain from one BIP-39 mnemonic:

```bash
ows wallet create --name "agent-treasury"
```

Output shows addresses for EVM, Solana, Bitcoin, Cosmos, Tron, TON, Sui, and Filecoin — all from one seed. The mnemonic is encrypted with AES-256-GCM and stored in `~/.ows/wallets/`.

In code:

```javascript
import { createWallet } from "@open-wallet-standard/core";
const wallet = createWallet("agent-treasury");
// wallet.accounts → [{chainId: "eip155:1", address: "0x...", derivationPath: "m/44'/60'/0'/0/0"}, ...]
```

```python
from ows import create_wallet
wallet = create_wallet("agent-treasury")
```

### 2. Sign Messages and Transactions

```bash
# Sign a message (EVM — uses EIP-191)
ows sign message --wallet agent-treasury --chain evm --message "hello world"

# Sign a raw transaction
ows sign tx --wallet agent-treasury --chain base --tx "02f8..."
```

In code:

```javascript
import { signMessage, signTransaction, signAndSend } from "@open-wallet-standard/core";

const sig = signMessage("agent-treasury", "evm", "hello world");
// sig.signature → hex string, sig.recoveryId → 0 or 1

const txSig = signTransaction("agent-treasury", "base", "02f8...");

// Sign AND broadcast (returns tx hash)
const result = signAndSend("agent-treasury", "base", "02f8...");
// result.txHash → "0xabc..."
```

```python
from ows import sign_message, sign_transaction, sign_and_send

sig = sign_message("agent-treasury", "evm", "hello world")
tx = sign_and_send("agent-treasury", "base", "02f8...")
```

### 3. Set Up Agent Access (The Killer Feature)

This is what makes OWS special for agents. Instead of giving an agent a raw private key, you create a **scoped API token** with policies that constrain what the agent can do. The agent never sees the private key.

**Step 1 — Define a policy:**

```bash
cat > policy.json << 'EOF'
{
  "id": "agent-limits",
  "name": "Base chain only, expires end of year",
  "version": 1,
  "created_at": "2026-01-01T00:00:00Z",
  "rules": [
    { "type": "allowed_chains", "chain_ids": ["eip155:8453"] },
    { "type": "expires_at", "timestamp": "2026-12-31T23:59:59Z" }
  ],
  "action": "deny"
}
EOF
ows policy create --file policy.json
```

**Step 2 — Create an API key with the policy attached:**

```bash
ows key create --name "my-agent" --wallet agent-treasury --policy agent-limits
# => ows_key_a1b2c3d4...  (shown ONCE — save it)
```

**Step 3 — Agent uses the token to sign:**

The token goes where the passphrase would go. OWS detects the `ows_key_` prefix, evaluates all attached policies, and only signs if every policy allows it.

```bash
OWS_PASSPHRASE="ows_key_a1b2c3d4..." \
  ows sign tx --wallet agent-treasury --chain base --tx 0x02f8...
```

```javascript
const result = signTransaction("agent-treasury", "base", "02f8...", "ows_key_a1b2c3d4...");
```

```python
result = sign_transaction("agent-treasury", "base", "02f8...", passphrase="ows_key_a1b2c3d4...")
```

**Step 4 — Revoke when done:**

```bash
ows key revoke --id <key-id> --confirm
```

The token becomes useless immediately — the encrypted mnemonic copy is deleted.

### 4. Check Balances and Fund

```bash
# Check balance on a chain
ows fund balance --wallet agent-treasury --chain base

# Create a deposit address (auto-converts to USDC on target chain)
ows fund deposit --wallet agent-treasury --chain base
```

### 5. Pay for Services (x402)

OWS handles x402 payment flows automatically — when a server returns `402 Payment Required`, it signs and retries:

```bash
ows pay request "https://api.example.com/data" --wallet agent-treasury
ows pay discover --query "weather"  # Find x402-enabled services
```

### 6. Import and Export

```bash
# Import from mnemonic (reads from stdin — never as CLI argument)
echo "word1 word2 ..." | ows wallet import --name "imported" --mnemonic

# Import a private key
echo "4c0883a691..." | ows wallet import --name "from-evm" --private-key

# Import from Solana keypair
ows wallet import --name "sol-wallet" --format solana-keypair --file ~/.config/solana/id.json

# Export (interactive confirmation required)
ows wallet export --wallet agent-treasury
```

---

## Which Interface to Use

| Context | Use | Why |
|---------|-----|-----|
| Shell scripts, automation, quick operations | **CLI** (`ows`) | Fastest path, no code needed |
| Node.js / TypeScript applications | **Node.js SDK** (`@open-wallet-standard/core`) | In-process, native bindings, type-safe |
| Python applications or agents | **Python SDK** (`open-wallet-standard`) | In-process, native bindings, dict returns |
| MCP-compatible AI agents | **MCP server** | Standard agent protocol |
| Need all three | **`curl` installer** | Installs everything at once |

All interfaces share the same Rust core and the same `~/.ows/` vault. A wallet created via CLI is usable from the Node.js SDK and vice versa.

---

## Security Rules

These are non-negotiable when working with OWS:

1. **Never log or print private keys or mnemonics.** OWS handles encryption — you should never need to see raw key material. If you need to pass a key, use `echo "..." | ows wallet import` so it doesn't appear in shell history.

2. **Always use API tokens for agent access.** The owner passphrase bypasses all policies. Agents should get scoped tokens with appropriate policies, never the passphrase.

3. **Check balance before operating.** Before signing transactions, verify the wallet has funds on the target chain. If not, use `ows fund balance` and `ows fund deposit` to get funds there.

4. **Use policies to constrain agent scope.** At minimum, use `allowed_chains` to restrict which chains the agent can sign for, and `expires_at` for time-bounded access. For more complex needs, write custom executable policies.

5. **Revoke tokens when access is no longer needed.** `ows key revoke` instantly invalidates a token — no key rotation required.

6. **Audit trail exists at `~/.ows/logs/audit.jsonl`.** Every signing operation is logged. Check it when debugging.

---

## Supported Chains

OWS supports 9 chain families from a single mnemonic. Use CAIP-2 identifiers in code and policy files — shorthand aliases (`base`, `ethereum`, `solana`) work only in CLI context.

| Family | CAIP-2 Example | Shorthand |
|--------|---------------|-----------|
| EVM | `eip155:8453` (Base) | `base`, `ethereum`, `polygon`, `arbitrum`, `optimism` |
| Solana | `solana:5eykt4UsFv8P8NJdTREpY1vzqKqZKvdp` | `solana` |
| Bitcoin | `bip122:000000000019d6689c085ae165831e93` | `bitcoin` |
| Cosmos | `cosmos:cosmoshub-4` | `cosmos` |
| Tron | `tron:mainnet` | `tron` |
| TON | `ton:mainnet` | `ton` |
| Sui | `sui:mainnet` | `sui` |
| Filecoin | `fil:mainnet` | `filecoin` |
| Spark | `spark:mainnet` | `spark` |

For the full chain reference (all networks, derivation paths, address formats), see [`references/chains.md`](references/chains.md).

---

## Policy Quick Reference

Two types of policies:

**Declarative rules** (evaluated in-process, microseconds):
- `allowed_chains` — restrict which chain IDs can be signed for
- `expires_at` — time-bound access

**Custom executable policies** (any logic — simulation, spending limits, allowlists):
- Write a script that reads `PolicyContext` from stdin and writes `{"allow": true/false}` to stdout
- Executables run only if declarative rules pass first (fast pre-filter)

All policies attached to an API key are AND-combined — every policy must allow for signing to proceed. Default-deny on any failure.

For the full policy engine reference (PolicyContext schema, executable protocol, examples), see [`references/policies.md`](references/policies.md).

---

## Deep-Dive References

Read these when you need specifics beyond the workflows above:

| Reference | When to read |
|-----------|-------------|
| [`references/cli.md`](references/cli.md) | You need the full CLI command reference — every command, flag, example |
| [`references/sdk-node.md`](references/sdk-node.md) | You're writing JavaScript/TypeScript and need function signatures and types |
| [`references/sdk-python.md`](references/sdk-python.md) | You're writing Python and need the API reference |
| [`references/policies.md`](references/policies.md) | You need to create or debug policies, understand the agent access model |
| [`references/chains.md`](references/chains.md) | You need chain IDs, derivation paths, address formats, or network details |

---

## Architecture at a Glance

```
Agent / CLI / App
       |
       |  OWS Interface (SDK / CLI / MCP)
       v
+---------------------+
|    Access Layer      |     1. Agent calls ows.sign()
|  +-----------------+ |     2. Policy engine evaluates constraints
|  | Policy Engine   | |     3. Key decrypted in hardened memory
|  | (pre-signing)   | |     4. Transaction signed
|  +--------+--------+ |     5. Key wiped immediately
|  +--------v--------+ |     6. Signature returned
|  | Signing Enclave | |
|  | (isolated)      | |     The agent NEVER sees the private key.
|  +--------+--------+ |
|  +--------v--------+ |
|  | Wallet Vault    | |
|  | ~/.ows/wallets/ | |
|  +-----------------+ |
+---------------------+
```

Keys are encrypted with AES-256-GCM at rest, decrypted only during signing (in hardened memory with `mlock()`), and wiped immediately after. The audit log at `~/.ows/logs/audit.jsonl` records every operation.
