# Policy Engine Reference

This is the complete reference for the OWS policy engine — the access model, API key cryptography, agent signing flow, policy file format, executable policy protocol, and practical examples.

---

## Access Model

OWS uses a two-tier access model. Every call that touches key material takes a `credential` parameter (passphrase or API token). The credential type determines the access path:

| Tier | Credential | Access | Policy checks | Key decryption |
|------|-----------|--------|---------------|----------------|
| **Owner** | Passphrase (string) | Full, unrestricted | None | scrypt(passphrase) |
| **Agent** | `ows_key_<64 hex>` token | Scoped by policy | All attached policies evaluated (AND semantics) | HKDF(token) |

The owner passphrase bypasses every policy. It exists for human operators. Agents should never receive the passphrase — they get scoped API tokens instead.

### Flow Diagram

```
sign_transaction(wallet, chain, tx, credential)
                                       |
                          +------------+------------+
                          |                         |
                     passphrase                ows_key_...
                          |                         |
                     owner mode                agent mode
                     no policy                 policies enforced
                          |                         |
                   scrypt decrypt             evaluate all policies
                          |                    (AND, short-circuit)
                          |                         |
                          |                 +-------+-------+
                          |                 |               |
                          |              ALLOWED         DENIED
                          |                 |               |
                          |           HKDF decrypt    POLICY_DENIED
                          |                 |         (key material
                          |                 |          never touched)
                          +--------+--------+
                                   |
                              HD-derive key
                                   |
                                 sign
                                   |
                              zeroize key
                                   |
                            return signature
```

---

## API Key Cryptography

When an owner creates an API key, OWS performs a cryptographic handoff so the agent can decrypt the mnemonic with its token alone, without ever knowing the passphrase.

### Key Creation Steps

1. Owner authenticates with passphrase.
2. OWS decrypts the wallet mnemonic using `scrypt(passphrase)`.
3. OWS generates a 256-bit random token: `ows_key_<64 hex chars>`.
4. OWS generates a random 32-byte salt.
5. OWS derives an encryption key: `HKDF-SHA256(ikm=token, salt=salt, info="ows-api-key-v1", dklen=32)`.
6. OWS re-encrypts the mnemonic under the HKDF-derived key (AES-256-GCM).
7. OWS computes `token_hash = SHA-256(token)` and stores it alongside the encrypted mnemonic copy, salt, policy IDs, wallet IDs, and metadata.
8. The raw token is displayed once. OWS does not retain it.

### Token Format

```
ows_key_a1b2c3d4e5f6...  (ows_key_ prefix + 64 hex characters = 256 bits)
```

### What Gets Stored on Disk

The API key file at `~/.ows/keys/<key-id>.json` contains:

| Field | Description |
|-------|-------------|
| `id` | Unique key identifier |
| `name` | Human-readable label |
| `token_hash` | SHA-256 of the raw token (hex). Used for lookup. |
| `salt` | Random 32 bytes (hex). Used in HKDF derivation. |
| `encrypted_mnemonic` | Mnemonic re-encrypted under HKDF(token). |
| `wallet_ids` | Array of wallet IDs this key can access. |
| `policy_ids` | Array of policy IDs evaluated on every signing request. |
| `expires_at` | Optional ISO-8601 expiration timestamp. |
| `created_at` | ISO-8601 creation timestamp. |

The raw token is **not** stored. Only `token_hash` is stored for lookup.

### Revocation

```bash
ows key revoke --id <key-id> --confirm
```

Revocation deletes the key file. The encrypted mnemonic copy is gone. Even if an attacker has the raw token, there is nothing left to decrypt.

---

## Agent Signing Flow

When an agent calls any signing function with an `ows_key_` credential, OWS executes the following 13-step flow:

```
 1.  Detect ows_key_ prefix on credential         → enter agent mode
 2.  SHA-256(token)                                → compute token_hash
 3.  Look up key file by token_hash                → load key metadata
 4.  Check expires_at                              → reject if expired
 5.  Verify wallet in key's wallet_ids scope       → reject if not scoped
 6.  Load policies from key's policy_ids           → resolve policy files
 7.  Build PolicyContext                            → assemble request context
 8.  Evaluate all policies (AND, short-circuit)    → first deny stops evaluation
 9.  If denied → return POLICY_DENIED error        → key material never touched
10.  HKDF-SHA256(token, salt, "ows-api-key-v1")   → derive decryption key
11.  Decrypt mnemonic, HD-derive chain key         → chain-specific private key
12.  Sign transaction                              → produce signature
13.  Zeroize all key material from memory          → mlock'd pages wiped
```

Steps 1-9 happen **before** any key material is decrypted. If a policy denies the request, the encrypted mnemonic is never touched. This is a critical security property: a compromised policy executable cannot extract key material because keys are only decrypted after all policies pass.

---

## Declarative Rules

Declarative rules are evaluated in-process in microseconds. They require no external calls, no subprocesses, and no I/O.

### `allowed_chains`

Restricts which chain IDs the agent can sign for.

```json
{
  "type": "allowed_chains",
  "chain_ids": ["eip155:8453", "eip155:84532"]
}
```

The `chain_ids` array uses CAIP-2 identifiers. If the signing request's `chain_id` is not in the list, the policy denies.

### `expires_at`

Time-bounds the policy. After the timestamp, the policy denies all requests.

```json
{
  "type": "expires_at",
  "timestamp": "2026-04-01T00:00:00Z"
}
```

The `timestamp` field is ISO-8601 in UTC. Comparison uses the system clock at evaluation time.

### Unknown Rule Types

If OWS encounters a rule with an unrecognized `type`, it **denies** the request. This prevents a misconfigured policy from silently allowing operations the author intended to restrict.

---

## Custom Executable Policies

For logic beyond declarative rules — transaction simulation, spending limits, allowlists, external approvals — use an executable policy. An executable is any program that follows the stdin/stdout protocol below.

### Protocol

```
echo '<PolicyContext JSON>' | /path/to/policy-executable
```

1. OWS spawns the executable as a child process.
2. OWS writes the full `PolicyContext` as a JSON object to the executable's **stdin**, then closes stdin.
3. The executable must write a `PolicyResult` JSON object to **stdout**.
4. OWS reads stdout and parses the result.

### Timeout and Failure Semantics

| Condition | Result |
|-----------|--------|
| Exit code 0, valid JSON on stdout | Use the `PolicyResult` as-is |
| Non-zero exit code | **Deny** (regardless of stdout) |
| Exit code 0, invalid JSON on stdout | **Deny** |
| Execution exceeds 5 seconds | **Deny** (process killed) |
| Executable not found at path | **Deny** |
| Unknown declarative rule type | **Deny** |

Every failure mode defaults to deny. There is no way for a broken or missing policy to accidentally allow a transaction.

### Evaluation Order Within a Policy

A single policy can have both declarative rules and an executable. They are evaluated in a specific order:

1. **Declarative rules first** (fast pre-filter).
2. If any declarative rule denies, the executable is **skipped** entirely (no process spawned).
3. If all declarative rules allow, the executable is **spawned**.
4. **Both** must allow for the policy to allow.

This ordering means declarative rules act as a cheap guard. If a request fails chain restrictions, OWS never pays the cost of spawning a subprocess.

---

## Policy File Format

Policy files live at `~/.ows/policies/<id>.json`. The full schema:

```json
{
  "id": "unique-policy-id",
  "name": "Human-readable policy name",
  "version": 1,
  "created_at": "2026-01-15T00:00:00Z",
  "rules": [
    { "type": "allowed_chains", "chain_ids": ["eip155:8453"] },
    { "type": "expires_at", "timestamp": "2026-12-31T23:59:59Z" }
  ],
  "executable": "/path/to/policy-executable",
  "config": {
    "rpc_url": "https://mainnet.base.org",
    "max_value_wei": "1000000000000000000"
  },
  "action": "deny"
}
```

### Field Reference

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `id` | string | Yes | Unique identifier. Used in API key `policy_ids` arrays. |
| `name` | string | Yes | Human-readable name for display and logging. |
| `version` | integer | Yes | Schema version. Currently `1`. |
| `created_at` | string | Yes | ISO-8601 creation timestamp. |
| `rules` | array | Conditional | Array of declarative rule objects. Required if `executable` is null. |
| `executable` | string or null | Conditional | Absolute path to the executable. Required if `rules` is empty or null. |
| `config` | object or null | No | Arbitrary configuration object passed to the executable via `PolicyContext`. |
| `action` | string | Yes | Action on policy violation. Currently always `"deny"`. |

A policy **must** have at least one of `rules` or `executable` (or both). A policy with neither is invalid and will cause all signing requests to be denied.

---

## PolicyContext Schema

The `PolicyContext` is the full request context evaluated by policies. For declarative rules, OWS evaluates the fields internally. For executable policies, the entire context is serialized as JSON and written to stdin.

```json
{
  "chain_id": "eip155:8453",
  "wallet_id": "agent-treasury",
  "api_key_id": "key-abc123",
  "transaction": {
    "to": "0x833589fCD6eDb6E08f4c7C32D4f71b54bdA02913",
    "value": "1000000000000000000",
    "raw_hex": "02f8...",
    "data": "0xa9059cbb..."
  },
  "spending": {
    "daily_total": "2500000000000000000",
    "date": "2026-03-24"
  },
  "timestamp": "2026-03-24T14:30:00Z",
  "policy_config": {
    "rpc_url": "https://mainnet.base.org",
    "max_value_wei": "1000000000000000000"
  }
}
```

### Field Reference

| Field | Type | Description |
|-------|------|-------------|
| `chain_id` | string | CAIP-2 chain identifier for the signing request. |
| `wallet_id` | string | ID of the wallet being signed with. |
| `api_key_id` | string | ID of the API key making the request. |
| `transaction.to` | string | Destination address. |
| `transaction.value` | string | Native token value in wei (decimal string). |
| `transaction.raw_hex` | string | Full raw transaction hex. |
| `transaction.data` | string | Transaction calldata (hex). Empty string for simple transfers. |
| `spending.daily_total` | string | Total value signed today across all transactions (decimal wei string). |
| `spending.date` | string | Current date in `YYYY-MM-DD` format. |
| `timestamp` | string | ISO-8601 timestamp of the signing request. |
| `policy_config` | object or null | The `config` object from the policy file, if present. Injected only when the policy defines a `config` field. Absent otherwise. |

The `policy_config` field lets you write a single reusable executable and parameterize it per-policy. For example, one executable can enforce spending limits with different thresholds depending on which policy references it.

---

## PolicyResult

The executable must write a JSON object to stdout with this shape:

### Allow

```json
{
  "allow": true
}
```

### Deny

```json
{
  "allow": false,
  "reason": "Transaction value exceeds daily spending limit of 1 ETH"
}
```

The `reason` field is optional when `allow` is `true`, but should always be provided when `allow` is `false`. The reason is included in the `POLICY_DENIED` error returned to the caller and logged in the audit trail.

---

## Security Properties

| Scenario | Outcome |
|----------|---------|
| Token stolen, no disk access | Useless. The encrypted mnemonic copy lives on disk. Token alone cannot derive keys without the encrypted data and salt. |
| Disk access, no token | Cannot decrypt. The mnemonic copy is encrypted under HKDF(token). Without the token, the encryption key cannot be derived. |
| Passphrase changed by owner | API keys are **unaffected**. Each API key holds its own independently encrypted copy of the mnemonic. Changing the passphrase re-encrypts the owner copy, not the API key copies. |
| API key revoked | Encrypted mnemonic copy is **deleted** from disk. The token becomes useless because there is nothing left to decrypt. |
| Policy executable compromised | Cannot extract key material. Policies run before decryption (steps 1-9). Key material is only decrypted after all policies pass (step 10). |
| Multiple API keys, one revoked | Other keys are unaffected. Each key has its own independent encrypted mnemonic copy. |

---

## Practical Examples

### Example 1: Basic Chain Restriction

Create a policy that restricts an agent to Base mainnet and Base Sepolia only.

**Policy file** (`base-only.json`):

```json
{
  "id": "base-only",
  "name": "Base chains only",
  "version": 1,
  "created_at": "2026-03-24T00:00:00Z",
  "rules": [
    {
      "type": "allowed_chains",
      "chain_ids": ["eip155:8453", "eip155:84532"]
    }
  ],
  "executable": null,
  "config": null,
  "action": "deny"
}
```

**Register and attach**:

```bash
# Register the policy
ows policy create --file base-only.json

# Create an API key scoped to the policy and wallet
ows key create --name "base-agent" --wallet agent-treasury --policy base-only
# => ows_key_7f3a...  (save this)
```

**Agent signs on Base** (allowed):

```bash
OWS_PASSPHRASE="ows_key_7f3a..." \
  ows sign tx --wallet agent-treasury --chain base --tx 0x02f8...
# => Success: signature returned
```

**Agent tries Ethereum mainnet** (denied):

```bash
OWS_PASSPHRASE="ows_key_7f3a..." \
  ows sign tx --wallet agent-treasury --chain ethereum --tx 0x02f8...
# => Error: POLICY_DENIED — chain eip155:1 not in allowed_chains [eip155:8453, eip155:84532]
```

---

### Example 2: Time-Bounded Agent Access

Create a policy that combines chain restriction with a time expiration. The agent can only sign on Base until April 1, 2026.

**Policy file** (`base-temporary.json`):

```json
{
  "id": "base-temporary",
  "name": "Base only, expires April 2026",
  "version": 1,
  "created_at": "2026-03-24T00:00:00Z",
  "rules": [
    {
      "type": "allowed_chains",
      "chain_ids": ["eip155:8453", "eip155:84532"]
    },
    {
      "type": "expires_at",
      "timestamp": "2026-04-01T00:00:00Z"
    }
  ],
  "executable": null,
  "config": null,
  "action": "deny"
}
```

```bash
ows policy create --file base-temporary.json
ows key create --name "temp-agent" --wallet agent-treasury --policy base-temporary
# => ows_key_9e1c...
```

Both rules must pass. If the chain is wrong, denied. If the time has passed, denied. If both conditions are met, allowed.

After April 1, 2026, every request through this key is denied regardless of chain — the `expires_at` rule fails, and OWS short-circuits without checking `allowed_chains`.

---

### Example 3: Custom Simulation Policy

Create an executable policy that simulates transactions via `eth_call` before allowing them. If the simulation reverts, the transaction is denied before signing.

**Policy file** (`simulate-tx.json`):

```json
{
  "id": "simulate-tx",
  "name": "Simulate before signing",
  "version": 1,
  "created_at": "2026-03-24T00:00:00Z",
  "rules": [
    {
      "type": "allowed_chains",
      "chain_ids": ["eip155:8453"]
    }
  ],
  "executable": "/home/agent/.ows/scripts/simulate.py",
  "config": {
    "rpc_url": "https://mainnet.base.org",
    "from_address": "0xYourWalletAddress"
  },
  "action": "deny"
}
```

**Executable** (`/home/agent/.ows/scripts/simulate.py`):

```python
#!/usr/bin/env python3
"""
OWS executable policy: simulate a transaction via eth_call before allowing it.

Receives PolicyContext JSON on stdin, writes PolicyResult JSON to stdout.
"""

import json
import sys
import urllib.request

def main():
    # Read PolicyContext from stdin
    context = json.load(sys.stdin)

    tx = context["transaction"]
    config = context.get("policy_config", {})

    rpc_url = config.get("rpc_url")
    from_address = config.get("from_address")

    if not rpc_url or not from_address:
        json.dump({"allow": False, "reason": "Missing rpc_url or from_address in policy config"}, sys.stdout)
        return

    # Build eth_call payload
    call_params = {
        "from": from_address,
        "to": tx["to"],
        "value": hex(int(tx["value"])) if tx.get("value") and tx["value"] != "0" else "0x0",
        "data": tx.get("data", "0x"),
    }

    payload = json.dumps({
        "jsonrpc": "2.0",
        "method": "eth_call",
        "params": [call_params, "latest"],
        "id": 1,
    }).encode()

    req = urllib.request.Request(
        rpc_url,
        data=payload,
        headers={"Content-Type": "application/json"},
    )

    try:
        with urllib.request.urlopen(req, timeout=4) as resp:
            result = json.loads(resp.read())
    except Exception as e:
        json.dump({"allow": False, "reason": f"Simulation RPC call failed: {e}"}, sys.stdout)
        return

    # Check for revert
    if "error" in result:
        error_msg = result["error"].get("message", "unknown error")
        json.dump({"allow": False, "reason": f"Simulation reverted: {error_msg}"}, sys.stdout)
        return

    # Simulation succeeded
    json.dump({"allow": True}, sys.stdout)

if __name__ == "__main__":
    main()
```

Make the script executable:

```bash
chmod +x /home/agent/.ows/scripts/simulate.py
```

**Evaluation flow for this policy**:

1. Declarative rule `allowed_chains` is checked first. If the chain is not `eip155:8453`, the request is denied immediately and `simulate.py` is never spawned.
2. If the chain check passes, OWS spawns `simulate.py`, pipes the full `PolicyContext` (including `policy_config` with `rpc_url` and `from_address`) to stdin.
3. The script performs `eth_call` against the RPC. If the call reverts, it returns `{"allow": false, "reason": "..."}`.
4. If the simulation succeeds, it returns `{"allow": true}`.
5. Both declarative rules and executable must allow for signing to proceed.

---

### Example 4: Multi-Agent Setup

Create two API keys for two agents with different permissions. Agent A operates on Base, Agent B operates on Arbitrum. Both are time-bounded.

**Policy for Agent A** (`policy-agent-a.json`):

```json
{
  "id": "agent-a-base",
  "name": "Agent A: Base only, 30-day window",
  "version": 1,
  "created_at": "2026-03-24T00:00:00Z",
  "rules": [
    {
      "type": "allowed_chains",
      "chain_ids": ["eip155:8453", "eip155:84532"]
    },
    {
      "type": "expires_at",
      "timestamp": "2026-04-23T00:00:00Z"
    }
  ],
  "executable": null,
  "config": null,
  "action": "deny"
}
```

**Policy for Agent B** (`policy-agent-b.json`):

```json
{
  "id": "agent-b-arbitrum",
  "name": "Agent B: Arbitrum only, 30-day window",
  "version": 1,
  "created_at": "2026-03-24T00:00:00Z",
  "rules": [
    {
      "type": "allowed_chains",
      "chain_ids": ["eip155:42161"]
    },
    {
      "type": "expires_at",
      "timestamp": "2026-04-23T00:00:00Z"
    }
  ],
  "executable": null,
  "config": null,
  "action": "deny"
}
```

**Register policies and create keys**:

```bash
# Register both policies
ows policy create --file policy-agent-a.json
ows policy create --file policy-agent-b.json

# Create API key for Agent A — scoped to Base policy and the shared wallet
ows key create --name "agent-a" --wallet agent-treasury --policy agent-a-base
# => ows_key_aaa111...

# Create API key for Agent B — scoped to Arbitrum policy and the same wallet
ows key create --name "agent-b" --wallet agent-treasury --policy agent-b-arbitrum
# => ows_key_bbb222...
```

**Agent A signs on Base** (allowed):

```javascript
import { signTransaction } from "@open-wallet-standard/core";

const sig = signTransaction("agent-treasury", "base", "02f8...", "ows_key_aaa111...");
// Success
```

**Agent A tries Arbitrum** (denied):

```javascript
const sig = signTransaction("agent-treasury", "arbitrum", "02f8...", "ows_key_aaa111...");
// Error: POLICY_DENIED — chain eip155:42161 not in allowed_chains [eip155:8453, eip155:84532]
```

**Agent B signs on Arbitrum** (allowed):

```python
from ows import sign_transaction

sig = sign_transaction("agent-treasury", "arbitrum", "02f8...", passphrase="ows_key_bbb222...")
# Success
```

**Agent B tries Base** (denied):

```python
sig = sign_transaction("agent-treasury", "base", "02f8...", passphrase="ows_key_bbb222...")
# Error: POLICY_DENIED — chain eip155:8453 not in allowed_chains [eip155:42161]
```

**Revoking one agent does not affect the other**:

```bash
# Agent A's access is no longer needed
ows key revoke --id agent-a --confirm
# Agent A's encrypted mnemonic copy is deleted. Agent B is unaffected.
```

Each API key holds an independent encrypted copy of the mnemonic. Revoking Agent A's key deletes only Agent A's encrypted copy. Agent B continues to operate normally with its own key and policies.
