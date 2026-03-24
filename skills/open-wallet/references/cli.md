# OWS CLI Reference

Complete command reference for the `ows` CLI. Every command, every flag, practical examples.

Binary location: `~/.ows/bin/ows`

---

## Wallet Commands

### `ows wallet create`

Create a new wallet. Generates a BIP-39 mnemonic and derives addresses for all supported chains.

| Flag | Required | Default | Description |
|------|----------|---------|-------------|
| `--name` | Yes | — | Human-readable wallet name (must be unique) |
| `--show-mnemonic` | No | false | Print the mnemonic to stdout after creation |
| `--words` | No | `12` | Mnemonic word count: `12` or `24` |

```bash
# Create a wallet with default 12-word mnemonic
ows wallet create --name "agent-treasury"

# Create a 24-word wallet and display the mnemonic
ows wallet create --name "cold-storage" --words 24 --show-mnemonic
```

---

### `ows wallet import`

Import an existing wallet from a mnemonic phrase or private key. Reads sensitive material from stdin or environment variables — never pass secrets as CLI arguments.

| Flag | Required | Default | Description |
|------|----------|---------|-------------|
| `--name` | Yes | — | Human-readable wallet name |
| `--mnemonic` | No | — | Import from BIP-39 mnemonic (read from stdin) |
| `--private-key` | No | — | Import from raw private key (read from stdin). Generates all 8 chain accounts. |
| `--chain` | No | — | Restrict derivation to a specific chain |
| `--index` | No | `0` | Derivation index (BIP-44 address index) |

**Environment variables for private key import:**

| Variable | Description |
|----------|-------------|
| `OWS_SECP256K1_KEY` | Hex-encoded secp256k1 private key (EVM, Bitcoin, Cosmos, Tron, Filecoin) |
| `OWS_ED25519_KEY` | Hex-encoded Ed25519 private key (Solana, TON, Sui) |

```bash
# Import from mnemonic via stdin
echo "abandon abandon abandon abandon abandon abandon abandon abandon abandon abandon abandon about" \
  | ows wallet import --name "restored" --mnemonic

# Import from mnemonic, derive only Solana
echo "abandon abandon ..." | ows wallet import --name "sol-only" --mnemonic --chain solana

# Import from mnemonic at derivation index 3
echo "abandon abandon ..." | ows wallet import --name "acct-3" --mnemonic --index 3

# Import from private key via stdin (generates all 8 chain accounts)
echo "4c0883a6918...deadbeef" | ows wallet import --name "from-key" --private-key

# Import using environment variables
OWS_SECP256K1_KEY="4c0883a6..." OWS_ED25519_KEY="7b2274..." \
  ows wallet import --name "dual-curve" --private-key
```

---

### `ows wallet export`

Export wallet key material. Mnemonic wallets output the phrase; private-key wallets output JSON with both curve keys.

| Flag | Required | Default | Description |
|------|----------|---------|-------------|
| `--wallet` | Yes | — | Wallet name or ID to export |

Requires passphrase via `OWS_PASSPHRASE` env var or interactive prompt.

```bash
# Export interactively (prompts for passphrase)
ows wallet export --wallet agent-treasury

# Export non-interactively
OWS_PASSPHRASE="my-secret" ows wallet export --wallet agent-treasury

# Mnemonic wallet output:
# abandon abandon abandon abandon abandon abandon abandon abandon abandon abandon abandon about

# Private-key wallet output:
# {"secp256k1": "4c0883a6...", "ed25519": "7b2274..."}
```

---

### `ows wallet list`

List all wallets in the vault. No flags.

```bash
ows wallet list

# Example output:
# ID         NAME              TYPE       CHAINS  CREATED
# w_a1b2c3   agent-treasury    mnemonic   8       2026-01-15T10:30:00Z
# w_d4e5f6   imported-evm      key        8       2026-02-01T14:00:00Z
```

---

### `ows wallet info`

Display vault path and list of supported chains with their CAIP-2 identifiers.

```bash
ows wallet info

# Example output:
# Vault path: /Users/you/.ows
# Supported chains:
#   eip155:1         Ethereum
#   eip155:8453      Base
#   eip155:42161     Arbitrum One
#   eip155:10        Optimism
#   eip155:137       Polygon
#   solana:5eykt...  Solana
#   bip122:00000...  Bitcoin
#   cosmos:cosmo...  Cosmos Hub
#   tron:mainnet     Tron
#   ton:mainnet      TON
#   sui:mainnet      Sui
#   fil:mainnet      Filecoin
#   spark:mainnet    Spark
```

---

### `ows wallet delete`

Permanently delete a wallet and all associated keys.

| Flag | Required | Default | Description |
|------|----------|---------|-------------|
| `--id` | Yes | — | Wallet ID (from `ows wallet list`) |
| `--confirm` | No | false | Skip interactive confirmation prompt |

```bash
# Interactive deletion
ows wallet delete --id w_a1b2c3

# Non-interactive (scripts/CI)
ows wallet delete --id w_a1b2c3 --confirm
```

---

## Signing Commands

Passphrases and API tokens are supplied via the `OWS_PASSPHRASE` environment variable or an interactive prompt. When the value starts with `ows_key_`, OWS treats it as an agent API token and evaluates all attached policies before signing.

### `ows sign message`

Sign an arbitrary message.

| Flag | Required | Default | Description |
|------|----------|---------|-------------|
| `--wallet` | Yes | — | Wallet name or ID |
| `--chain` | Yes | — | Chain shorthand or CAIP-2 ID |
| `--message` | Yes | — | Message string to sign |
| `--encoding` | No | `utf8` | Message encoding: `utf8` or `hex` |
| `--typed-data` | No | — | EIP-712 typed data as JSON string (EVM only) |
| `--json` | No | false | Output result as JSON |

```bash
# Sign a UTF-8 message on EVM
ows sign message --wallet agent-treasury --chain evm --message "hello world"

# Sign a hex-encoded message
ows sign message --wallet agent-treasury --chain evm --message "68656c6c6f" --encoding hex

# Sign EIP-712 typed data
ows sign message --wallet agent-treasury --chain evm \
  --typed-data '{"types":{"EIP712Domain":[{"name":"name","type":"string"}],"Mail":[{"name":"contents","type":"string"}]},"primaryType":"Mail","domain":{"name":"Example"},"message":{"contents":"Hello"}}'

# Sign on Solana with JSON output
ows sign message --wallet agent-treasury --chain solana --message "verify me" --json

# Agent signs using API token
OWS_PASSPHRASE="ows_key_a1b2c3d4..." \
  ows sign message --wallet agent-treasury --chain evm --message "agent-signed"
```

---

### `ows sign tx`

Sign a raw transaction.

| Flag | Required | Default | Description |
|------|----------|---------|-------------|
| `--wallet` | Yes | — | Wallet name or ID |
| `--chain` | Yes | — | Chain shorthand or CAIP-2 ID |
| `--tx` | Yes | — | Hex-encoded raw transaction |
| `--json` | No | false | Output result as JSON |

```bash
# Sign an EVM transaction on Base
ows sign tx --wallet agent-treasury --chain base --tx "02f870820..."

# Sign with JSON output
ows sign tx --wallet agent-treasury --chain base --tx "02f870820..." --json

# Agent signs with scoped token
OWS_PASSPHRASE="ows_key_a1b2c3d4..." \
  ows sign tx --wallet agent-treasury --chain base --tx "02f870820..."

# Owner signs with passphrase (bypasses all policies)
OWS_PASSPHRASE="my-owner-passphrase" \
  ows sign tx --wallet agent-treasury --chain ethereum --tx "02f870820..."
```

---

## Policy Commands

Policies are declarative rule sets that constrain what agent API tokens can do. Rules within a policy are AND-combined — every rule must pass for the policy to allow signing.

### `ows policy create`

Create a new policy from a JSON file.

| Flag | Required | Default | Description |
|------|----------|---------|-------------|
| `--file` | Yes | — | Path to policy JSON file |

**Declarative rule types:**

| Rule Type | Fields | Description |
|-----------|--------|-------------|
| `allowed_chains` | `chain_ids` (string array) | Only allow signing on these CAIP-2 chain IDs |
| `expires_at` | `timestamp` (ISO-8601) | Policy expires after this timestamp |

A policy can also include an `executable` field pointing to a script for custom logic (simulation, spending limits, allowlists). Executables run only after declarative rules pass.

```bash
# Create a policy JSON file
cat > base-only-policy.json << 'EOF'
{
  "id": "base-only",
  "name": "Base chain only, expires 2026-12-31",
  "version": 1,
  "created_at": "2026-01-01T00:00:00Z",
  "rules": [
    { "type": "allowed_chains", "chain_ids": ["eip155:8453"] },
    { "type": "expires_at", "timestamp": "2026-12-31T23:59:59Z" }
  ],
  "action": "deny"
}
EOF

ows policy create --file base-only-policy.json

# Policy with multiple allowed chains
cat > multi-chain-policy.json << 'EOF'
{
  "id": "evm-chains",
  "name": "EVM mainnet chains only",
  "version": 1,
  "created_at": "2026-03-01T00:00:00Z",
  "rules": [
    {
      "type": "allowed_chains",
      "chain_ids": ["eip155:1", "eip155:8453", "eip155:42161", "eip155:10"]
    }
  ],
  "action": "deny"
}
EOF

ows policy create --file multi-chain-policy.json

# Policy with custom executable
cat > simulation-policy.json << 'EOF'
{
  "id": "simulate-first",
  "name": "Simulate transactions before signing",
  "version": 1,
  "created_at": "2026-03-01T00:00:00Z",
  "rules": [
    { "type": "allowed_chains", "chain_ids": ["eip155:8453"] }
  ],
  "executable": "/home/user/scripts/simulate-tx.sh",
  "action": "deny"
}
EOF

ows policy create --file simulation-policy.json
```

---

### `ows policy list`

List all policies. No flags.

```bash
ows policy list

# Example output:
# ID              NAME                                RULES  CREATED
# base-only       Base chain only, expires 2026-12-31 2      2026-01-01T00:00:00Z
# evm-chains      EVM mainnet chains only             1      2026-03-01T00:00:00Z
```

---

### `ows policy show`

Display full details of a policy.

| Flag | Required | Default | Description |
|------|----------|---------|-------------|
| `--id` | Yes | — | Policy ID |

```bash
ows policy show --id base-only

# Example output:
# Policy: base-only
# Name: Base chain only, expires 2026-12-31
# Version: 1
# Created: 2026-01-01T00:00:00Z
# Default action: deny
# Rules:
#   1. allowed_chains: ["eip155:8453"]
#   2. expires_at: 2026-12-31T23:59:59Z
```

---

### `ows policy delete`

Delete a policy. Keys referencing this policy will lose this constraint.

| Flag | Required | Default | Description |
|------|----------|---------|-------------|
| `--id` | Yes | — | Policy ID |
| `--confirm` | No | false | Skip interactive confirmation prompt |

```bash
ows policy delete --id base-only --confirm
```

---

## Key Commands (Agent Access)

API keys give agents scoped access to wallets. The agent uses the token in place of a passphrase. OWS evaluates all attached policies before signing.

### `ows key create`

Create a new API key. Returns a raw token (`ows_key_...`) that is shown only once.

| Flag | Required | Default | Description |
|------|----------|---------|-------------|
| `--name` | Yes | — | Human-readable key name |
| `--wallet` | No (repeatable) | — | Wallet(s) the key can access. Repeat for multiple wallets. |
| `--policy` | No (repeatable) | — | Policy/policies to attach. Repeat for multiple policies. |
| `--expires-at` | No | — | ISO-8601 expiration timestamp for the key itself |

```bash
# Create a key for one wallet with one policy
ows key create --name "my-agent" --wallet agent-treasury --policy base-only

# Output:
# Key created: k_x9y8z7
# Token: ows_key_a1b2c3d4e5f6g7h8i9j0...
# WARNING: This token is shown ONCE. Save it now.

# Create a key for multiple wallets with multiple policies
ows key create --name "multi-agent" \
  --wallet agent-treasury \
  --wallet hot-wallet \
  --policy base-only \
  --policy evm-chains

# Create a key with an expiration
ows key create --name "temp-agent" \
  --wallet agent-treasury \
  --policy base-only \
  --expires-at "2026-06-30T23:59:59Z"
```

---

### `ows key list`

List all API keys. Tokens are never displayed.

```bash
ows key list

# Example output:
# ID         NAME         WALLETS          POLICIES    EXPIRES             CREATED
# k_x9y8z7   my-agent     agent-treasury   base-only   —                   2026-01-15T10:30:00Z
# k_m3n4o5   temp-agent   agent-treasury   base-only   2026-06-30T23:59Z   2026-03-01T14:00:00Z
```

---

### `ows key revoke`

Revoke an API key. The token becomes immediately unusable and the encrypted key copy is deleted.

| Flag | Required | Default | Description |
|------|----------|---------|-------------|
| `--id` | Yes | — | Key ID (from `ows key list`) |
| `--confirm` | No | false | Skip interactive confirmation prompt |

```bash
# Revoke interactively
ows key revoke --id k_x9y8z7

# Revoke non-interactively
ows key revoke --id k_x9y8z7 --confirm
```

---

## Payment Commands (x402)

OWS handles the x402 payment protocol automatically. When a server returns `402 Payment Required`, OWS signs the payment and retries the request.

### `ows pay request`

Make an HTTP request with automatic x402 payment handling.

| Flag | Required | Default | Description |
|------|----------|---------|-------------|
| `--wallet` | Yes | — | Wallet name or ID to pay from |
| `--method` | No | `GET` | HTTP method: `GET`, `POST`, `PUT`, `DELETE`, `PATCH` |
| `--body` | No | — | Request body as JSON string |
| `--no-passphrase` | No | false | Skip passphrase prompt (use with API tokens in `OWS_PASSPHRASE`) |

The first positional argument is the URL.

```bash
# Simple GET request with x402 payment
ows pay request "https://api.example.com/data" --wallet agent-treasury

# POST request with body
ows pay request "https://api.example.com/submit" \
  --wallet agent-treasury \
  --method POST \
  --body '{"query": "weather in NYC"}'

# Agent makes a paid request using API token
OWS_PASSPHRASE="ows_key_a1b2c3d4..." \
  ows pay request "https://api.example.com/data" \
  --wallet agent-treasury \
  --no-passphrase
```

---

### `ows pay discover`

Discover x402-enabled services.

| Flag | Required | Default | Description |
|------|----------|---------|-------------|
| `--query` | No | — | Search query string |
| `--limit` | No | `10` | Maximum number of results |
| `--offset` | No | `0` | Pagination offset |

```bash
# Discover services
ows pay discover --query "weather"

# Paginate results
ows pay discover --query "AI" --limit 20 --offset 40
```

---

## Funding Commands

### `ows fund deposit`

Create a deposit address. Funds sent to this address are auto-converted to USDC on the target chain.

| Flag | Required | Default | Description |
|------|----------|---------|-------------|
| `--wallet` | Yes | — | Wallet name or ID |
| `--chain` | No | `base` | Target chain for the deposit |

```bash
# Create a deposit address on Base (default)
ows fund deposit --wallet agent-treasury

# Create a deposit address on Ethereum
ows fund deposit --wallet agent-treasury --chain ethereum
```

---

### `ows fund balance`

Check the balance of a wallet on a specific chain.

| Flag | Required | Default | Description |
|------|----------|---------|-------------|
| `--wallet` | Yes | — | Wallet name or ID |
| `--chain` | Yes | — | Chain to check balance on |

```bash
# Check balance on Base
ows fund balance --wallet agent-treasury --chain base

# Check balance on Solana
ows fund balance --wallet agent-treasury --chain solana

# Check balance on Ethereum mainnet using CAIP-2
ows fund balance --wallet agent-treasury --chain eip155:1
```

---

## Mnemonic Commands

Standalone mnemonic utilities, independent of the wallet vault.

### `ows mnemonic generate`

Generate a new BIP-39 mnemonic phrase without creating a wallet.

| Flag | Required | Default | Description |
|------|----------|---------|-------------|
| `--words` | No | `12` | Word count: `12` or `24` |

```bash
# Generate a 12-word mnemonic
ows mnemonic generate

# Generate a 24-word mnemonic
ows mnemonic generate --words 24
```

---

### `ows mnemonic derive`

Derive addresses from a mnemonic without importing into the vault. Reads the mnemonic from the `OWS_MNEMONIC` environment variable or stdin.

| Flag | Required | Default | Description |
|------|----------|---------|-------------|
| `--chain` | No | — | Derive only for a specific chain |

```bash
# Derive all chain addresses from env var
OWS_MNEMONIC="abandon abandon abandon abandon abandon abandon abandon abandon abandon abandon abandon about" \
  ows mnemonic derive

# Derive only Solana address from stdin
echo "abandon abandon abandon abandon abandon abandon abandon abandon abandon abandon abandon about" \
  | ows mnemonic derive --chain solana

# Derive only EVM address
OWS_MNEMONIC="abandon abandon ..." ows mnemonic derive --chain evm
```

---

## System Commands

### `ows update`

Update the OWS CLI to the latest version.

| Flag | Required | Default | Description |
|------|----------|---------|-------------|
| `--force` | No | false | Force reinstall even if already on latest version |

```bash
# Update to latest
ows update

# Force reinstall
ows update --force
```

---

### `ows uninstall`

Remove the OWS CLI.

| Flag | Required | Default | Description |
|------|----------|---------|-------------|
| `--purge` | No | false | Also remove `~/.ows/` directory (vault, keys, policies, logs) |

```bash
# Remove CLI binary only (vault preserved)
ows uninstall

# Remove everything including vault data
ows uninstall --purge
```

---

## End-to-End Example: Agent Access Lifecycle

A complete walkthrough: create a wallet, define a policy, issue an API key to an agent, observe policy enforcement, and clean up.

```bash
# --------------------------------------------------
# Step 1: Create a wallet
# --------------------------------------------------
ows wallet create --name "ops-wallet"
# => Wallet created: w_abc123
# => Addresses derived for 8 chains

# --------------------------------------------------
# Step 2: Fund the wallet on Base
# --------------------------------------------------
ows fund deposit --wallet ops-wallet --chain base
# => Deposit address: 0x... (send funds here, auto-converts to USDC)

ows fund balance --wallet ops-wallet --chain base
# => 100.00 USDC

# --------------------------------------------------
# Step 3: Create a policy — Base chain only, expires end of quarter
# --------------------------------------------------
cat > agent-policy.json << 'EOF'
{
  "id": "q1-base-only",
  "name": "Q1 2026 - Base chain only",
  "version": 1,
  "created_at": "2026-01-01T00:00:00Z",
  "rules": [
    { "type": "allowed_chains", "chain_ids": ["eip155:8453"] },
    { "type": "expires_at", "timestamp": "2026-03-31T23:59:59Z" }
  ],
  "action": "deny"
}
EOF

ows policy create --file agent-policy.json
# => Policy created: q1-base-only

# --------------------------------------------------
# Step 4: Create an API key for the agent
# --------------------------------------------------
ows key create --name "trading-bot" --wallet ops-wallet --policy q1-base-only
# => Key created: k_def456
# => Token: ows_key_t0k3nv4lu3...
# => WARNING: This token is shown ONCE. Save it now.

# --------------------------------------------------
# Step 5: Agent signs on Base (ALLOWED)
# --------------------------------------------------
OWS_PASSPHRASE="ows_key_t0k3nv4lu3..." \
  ows sign tx --wallet ops-wallet --chain base --tx "02f870820..."
# => Signature: 0x3045...
# => Policy q1-base-only: PASS (chain eip155:8453 allowed, not expired)

# --------------------------------------------------
# Step 6: Agent attempts to sign on Ethereum (DENIED)
# --------------------------------------------------
OWS_PASSPHRASE="ows_key_t0k3nv4lu3..." \
  ows sign tx --wallet ops-wallet --chain ethereum --tx "02f870820..."
# => ERROR: Policy denied — chain eip155:1 not in allowed_chains ["eip155:8453"]

# --------------------------------------------------
# Step 7: Owner signs on Ethereum with passphrase (BYPASSES policies)
# --------------------------------------------------
OWS_PASSPHRASE="owner-passphrase" \
  ows sign tx --wallet ops-wallet --chain ethereum --tx "02f870820..."
# => Signature: 0x3046...
# => (Owner passphrase bypasses all policy checks)

# --------------------------------------------------
# Step 8: Revoke the agent's key
# --------------------------------------------------
ows key revoke --id k_def456 --confirm
# => Key k_def456 revoked. Token is now invalid.

# Verify revocation — agent can no longer sign
OWS_PASSPHRASE="ows_key_t0k3nv4lu3..." \
  ows sign tx --wallet ops-wallet --chain base --tx "02f870820..."
# => ERROR: Invalid or revoked API token
```

---

## File Layout

The `~/.ows/` directory structure:

```
~/.ows/
  bin/
    ows                     # CLI binary
  wallets/
    <wallet-id>.enc         # AES-256-GCM encrypted wallet (mnemonic or key material)
  keys/
    <key-id>.enc            # Encrypted agent key copies (deleted on revoke)
  policies/
    <policy-id>.json        # Policy definitions
  logs/
    audit.jsonl             # Append-only audit log (every sign operation)
  config.toml               # Global configuration
```

**Key points:**
- Wallet files (`.enc`) are AES-256-GCM encrypted at rest, decrypted only in hardened memory (`mlock()`) during signing, and wiped immediately after.
- Agent key files are encrypted copies scoped to a specific token. Revoking a key deletes its `.enc` file.
- The audit log at `logs/audit.jsonl` records every signing operation with timestamp, wallet, chain, key used, and policy evaluation results.
- All interfaces (CLI, Node.js SDK, Python SDK, MCP server) share this same vault directory.
