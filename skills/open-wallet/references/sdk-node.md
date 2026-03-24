# Node.js SDK Reference — `@open-wallet-standard/core`

Complete API reference for the Open Wallet Standard Node.js SDK.

The package ships native bindings via NAPI (Rust core runs in-process). Prebuilt binaries are included for macOS (arm64, x64) and Linux (x64, arm64) — no Rust toolchain required.

---

## Install

```bash
npm install @open-wallet-standard/core
```

All functions are named exports:

```typescript
import {
  generateMnemonic,
  deriveAddress,
  createWallet,
  listWallets,
  getWallet,
  deleteWallet,
  renameWallet,
  exportWallet,
  importWalletMnemonic,
  importWalletPrivateKey,
  signMessage,
  signTypedData,
  signTransaction,
  signAndSend,
  createPolicy,
  listPolicies,
  getPolicy,
  deletePolicy,
  createApiKey,
  listApiKeys,
  revokeApiKey,
} from "@open-wallet-standard/core";
```

---

## Types

### AccountInfo

```typescript
interface AccountInfo {
  /** CAIP-2 chain identifier, e.g. "eip155:1", "solana:5eykt4UsFv8P8NJdTREpY1vzqKqZKvdp" */
  chainId: string;
  /** On-chain address in the native format for the chain */
  address: string;
  /** BIP-44 (or chain-specific) derivation path, e.g. "m/44'/60'/0'/0/0" */
  derivationPath: string;
}
```

### WalletInfo

```typescript
interface WalletInfo {
  /** Unique wallet identifier (UUID v4) */
  id: string;
  /** Human-readable wallet name */
  name: string;
  /** One account per supported chain */
  accounts: AccountInfo[];
  /** Wallet creation timestamp (ISO 8601) */
  createdAt: string;
}
```

### SignResult

```typescript
interface SignResult {
  /** Hex-encoded signature */
  signature: string;
  /** ECDSA recovery ID (0 or 1). Present for secp256k1 signatures (EVM, Bitcoin, Cosmos, Tron, Filecoin). Absent for Ed25519 signatures (Solana, Sui, TON). */
  recoveryId?: number;
}
```

### SendResult

```typescript
interface SendResult {
  /** Transaction hash returned by the network */
  txHash: string;
}
```

### ApiKeyResult

```typescript
interface ApiKeyResult {
  /** The API key token (shown once — store it securely). Prefixed with "ows_key_". */
  token: string;
  /** Unique identifier for this API key (use for revocation) */
  id: string;
  /** Human-readable name */
  name: string;
}
```

---

## Mnemonic

### generateMnemonic

Generate a BIP-39 mnemonic phrase.

```typescript
function generateMnemonic(words?: number): string;
```

| Param | Type | Default | Description |
|-------|------|---------|-------------|
| `words` | `number` | `12` | Number of words. Must be 12, 15, 18, 21, or 24. |

**Returns:** Space-separated mnemonic phrase.

```typescript
import { generateMnemonic } from "@open-wallet-standard/core";

const mnemonic12 = generateMnemonic();
// "abandon abandon abandon abandon abandon abandon abandon abandon abandon abandon abandon about"

const mnemonic24 = generateMnemonic(24);
// 24-word mnemonic for higher entropy (256-bit)
```

---

### deriveAddress

Derive a single address from a mnemonic for a specific chain.

```typescript
function deriveAddress(mnemonic: string, chain: string, index?: number): string;
```

| Param | Type | Default | Description |
|-------|------|---------|-------------|
| `mnemonic` | `string` | — | BIP-39 mnemonic phrase |
| `chain` | `string` | — | Chain family: `"evm"`, `"solana"`, `"sui"`, `"bitcoin"`, `"cosmos"`, `"tron"`, `"ton"`, `"spark"`, `"filecoin"` |
| `index` | `number` | `0` | Account index in the derivation path |

**Returns:** Address string in the native format for the chain.

```typescript
import { generateMnemonic, deriveAddress } from "@open-wallet-standard/core";

const mnemonic = generateMnemonic();

const evmAddr = deriveAddress(mnemonic, "evm");
// "0x742d35Cc6634C0532925a3b844Bc9e7595f2bD18"

const solAddr = deriveAddress(mnemonic, "solana");
// "7EcDhSYGxXyscszYEp35KHN8vvw3svAuLKTzXwCFLtV"

const btcAddr = deriveAddress(mnemonic, "bitcoin");
// "bc1qw508d6qejxtdg4y5r3zarvary0c5xw7kv8f3t4"

// Derive from a different account index
const evmAddr1 = deriveAddress(mnemonic, "evm", 1);
```

---

## Wallet Management

### createWallet

Create a new wallet. Generates a fresh BIP-39 mnemonic, derives addresses for all supported chains, and stores the encrypted mnemonic in the vault.

```typescript
function createWallet(
  name: string,
  passphrase?: string,
  words?: number,
  vaultPath?: string
): WalletInfo;
```

| Param | Type | Default | Description |
|-------|------|---------|-------------|
| `name` | `string` | — | Human-readable wallet name (must be unique within the vault) |
| `passphrase` | `string` | `undefined` | Encryption passphrase. If omitted, the mnemonic is stored with a default passphrase. |
| `words` | `number` | `12` | Mnemonic word count (12, 15, 18, 21, or 24) |
| `vaultPath` | `string` | `"~/.ows"` | Custom vault directory |

**Returns:** `WalletInfo` with all derived accounts.

```typescript
import { createWallet } from "@open-wallet-standard/core";

// Minimal — create with defaults
const wallet = createWallet("agent-treasury");
console.log(wallet.id);        // "a1b2c3d4-..."
console.log(wallet.name);      // "agent-treasury"
console.log(wallet.accounts);  // 9 accounts (one per chain)
console.log(wallet.createdAt); // "2026-03-24T12:00:00.000Z"

// With passphrase and 24-word mnemonic
const secure = createWallet("high-value", "my-strong-passphrase", 24);

// With custom vault path (useful for testing)
const testWallet = createWallet("test-wallet", undefined, 12, "/tmp/test-vault");
```

---

### listWallets

List all wallets in the vault.

```typescript
function listWallets(vaultPath?: string): WalletInfo[];
```

| Param | Type | Default | Description |
|-------|------|---------|-------------|
| `vaultPath` | `string` | `"~/.ows"` | Custom vault directory |

**Returns:** Array of `WalletInfo`.

```typescript
import { listWallets } from "@open-wallet-standard/core";

const wallets = listWallets();
for (const w of wallets) {
  console.log(`${w.name} (${w.id}) — ${w.accounts.length} accounts`);
}
```

---

### getWallet

Retrieve a single wallet by name or ID.

```typescript
function getWallet(nameOrId: string, vaultPath?: string): WalletInfo;
```

| Param | Type | Default | Description |
|-------|------|---------|-------------|
| `nameOrId` | `string` | — | Wallet name or UUID |
| `vaultPath` | `string` | `"~/.ows"` | Custom vault directory |

**Returns:** `WalletInfo`.

**Throws:** Error if wallet not found.

```typescript
import { getWallet } from "@open-wallet-standard/core";

const wallet = getWallet("agent-treasury");

// Find the EVM address
const evmAccount = wallet.accounts.find((a) => a.chainId.startsWith("eip155:"));
console.log(evmAccount.address); // "0x..."

// Look up by ID
const same = getWallet(wallet.id);
```

---

### deleteWallet

Permanently delete a wallet from the vault. This removes the encrypted mnemonic — the action is irreversible.

```typescript
function deleteWallet(nameOrId: string, vaultPath?: string): void;
```

| Param | Type | Default | Description |
|-------|------|---------|-------------|
| `nameOrId` | `string` | — | Wallet name or UUID |
| `vaultPath` | `string` | `"~/.ows"` | Custom vault directory |

```typescript
import { deleteWallet } from "@open-wallet-standard/core";

deleteWallet("old-wallet");
```

---

### renameWallet

Rename an existing wallet.

```typescript
function renameWallet(nameOrId: string, newName: string, vaultPath?: string): void;
```

| Param | Type | Default | Description |
|-------|------|---------|-------------|
| `nameOrId` | `string` | — | Current wallet name or UUID |
| `newName` | `string` | — | New wallet name (must be unique within the vault) |
| `vaultPath` | `string` | `"~/.ows"` | Custom vault directory |

```typescript
import { renameWallet } from "@open-wallet-standard/core";

renameWallet("agent-treasury", "agent-treasury-v2");
```

---

### exportWallet

Export the mnemonic phrase (or both-curve key JSON) for a wallet. Use with extreme caution — the exported material grants full control.

```typescript
function exportWallet(nameOrId: string, passphrase?: string, vaultPath?: string): string;
```

| Param | Type | Default | Description |
|-------|------|---------|-------------|
| `nameOrId` | `string` | — | Wallet name or UUID |
| `passphrase` | `string` | `undefined` | Decryption passphrase (must match the one used at creation) |
| `vaultPath` | `string` | `"~/.ows"` | Custom vault directory |

**Returns:** The mnemonic phrase as a string for mnemonic-based wallets. For private-key-imported wallets with both curves, returns a JSON string containing `secp256k1Key` and `ed25519Key` fields.

```typescript
import { exportWallet } from "@open-wallet-standard/core";

// Export a mnemonic-based wallet
const mnemonic = exportWallet("agent-treasury", "my-passphrase");
// "abandon abandon abandon ..."

// Export a dual-curve private-key wallet
const keysJson = exportWallet("pk-wallet", "my-passphrase");
// '{"secp256k1Key":"4c0883a6...","ed25519Key":"3b6a27bc..."}'
```

---

## Import

### importWalletMnemonic

Import a wallet from an existing BIP-39 mnemonic phrase. Derives addresses for all supported chains.

```typescript
function importWalletMnemonic(
  name: string,
  mnemonic: string,
  passphrase?: string,
  index?: number,
  vaultPath?: string
): WalletInfo;
```

| Param | Type | Default | Description |
|-------|------|---------|-------------|
| `name` | `string` | — | Wallet name |
| `mnemonic` | `string` | — | BIP-39 mnemonic phrase (12, 15, 18, 21, or 24 words) |
| `passphrase` | `string` | `undefined` | Encryption passphrase |
| `index` | `number` | `0` | Account index for derivation |
| `vaultPath` | `string` | `"~/.ows"` | Custom vault directory |

**Returns:** `WalletInfo` with all derived accounts.

```typescript
import { importWalletMnemonic } from "@open-wallet-standard/core";

const wallet = importWalletMnemonic(
  "restored-wallet",
  "abandon abandon abandon abandon abandon abandon abandon abandon abandon abandon abandon about"
);

// With passphrase and non-zero index
const wallet2 = importWalletMnemonic(
  "second-account",
  "abandon abandon abandon abandon abandon abandon abandon abandon abandon abandon abandon about",
  "strong-passphrase",
  1
);
```

---

### importWalletPrivateKey

Import a wallet from a raw private key. Always generates accounts for all 8 chain families regardless of which key is provided.

```typescript
function importWalletPrivateKey(
  name: string,
  privateKeyHex: string,
  passphrase?: string,
  vaultPath?: string,
  chain?: string,
  secp256k1Key?: string,
  ed25519Key?: string
): WalletInfo;
```

| Param | Type | Default | Description |
|-------|------|---------|-------------|
| `name` | `string` | — | Wallet name |
| `privateKeyHex` | `string` | — | Hex-encoded private key |
| `passphrase` | `string` | `undefined` | Encryption passphrase |
| `vaultPath` | `string` | `"~/.ows"` | Custom vault directory |
| `chain` | `string` | `"evm"` | Chain hint to determine curve. `"evm"`, `"bitcoin"`, `"cosmos"`, `"tron"`, `"filecoin"` use secp256k1. `"solana"`, `"sui"`, `"ton"` use Ed25519. |
| `secp256k1Key` | `string` | `undefined` | Explicit hex-encoded secp256k1 private key (for dual-curve import) |
| `ed25519Key` | `string` | `undefined` | Explicit hex-encoded Ed25519 private key (for dual-curve import) |

**Returns:** `WalletInfo` with accounts on all chains.

**Curve selection logic:**

- The `chain` parameter determines which curve the `privateKeyHex` value is interpreted as.
- Default is `"evm"` (secp256k1). Set `chain` to `"solana"`, `"sui"`, or `"ton"` for Ed25519.
- To provide keys for both curves explicitly, use `secp256k1Key` and `ed25519Key`. When both are provided, `privateKeyHex` can be set to either one (it is overridden by the explicit keys).
- All 8 chain accounts are always generated. Chains whose curve key was not provided will show a zeroed/placeholder address.

```typescript
import { importWalletPrivateKey } from "@open-wallet-standard/core";

// Import an EVM private key (secp256k1 — default)
const evmWallet = importWalletPrivateKey(
  "from-metamask",
  "4c0883a69102937d6231471b5dbb6204fe512961708279f23efb02eb56a0c7af"
);

// Import a Solana private key (Ed25519)
const solWallet = importWalletPrivateKey(
  "from-phantom",
  "3b6a27bcceb6a42d62a3a8d02a6f0d73653215771de243a63ac048a18b59da29",
  undefined, // no passphrase
  undefined, // default vault
  "solana"   // Ed25519 curve
);

// Import with both curves for full cross-chain support
const dualWallet = importWalletPrivateKey(
  "full-import",
  "4c0883a69102937d6231471b5dbb6204fe512961708279f23efb02eb56a0c7af",
  "my-passphrase",
  undefined, // default vault
  undefined, // chain hint not needed when both keys are explicit
  "4c0883a69102937d6231471b5dbb6204fe512961708279f23efb02eb56a0c7af", // secp256k1
  "3b6a27bcceb6a42d62a3a8d02a6f0d73653215771de243a63ac048a18b59da29"  // ed25519
);
```

---

## Signing

### signMessage

Sign a plain-text or binary message. Uses the chain-appropriate signing scheme (EIP-191 for EVM, Ed25519 for Solana/Sui/TON, etc.).

```typescript
function signMessage(
  wallet: string,
  chain: string,
  message: string,
  passphrase?: string,
  encoding?: string,
  index?: number,
  vaultPath?: string
): SignResult;
```

| Param | Type | Default | Description |
|-------|------|---------|-------------|
| `wallet` | `string` | — | Wallet name or UUID |
| `chain` | `string` | — | Chain family: `"evm"`, `"solana"`, `"sui"`, `"bitcoin"`, `"cosmos"`, `"tron"`, `"ton"`, `"spark"`, `"filecoin"` |
| `message` | `string` | — | Message to sign |
| `passphrase` | `string` | `undefined` | Decryption passphrase or API key token (`ows_key_...`) |
| `encoding` | `string` | `"utf8"` | Message encoding: `"utf8"` or `"hex"` |
| `index` | `number` | `0` | Account index |
| `vaultPath` | `string` | `"~/.ows"` | Custom vault directory |

**Returns:** `SignResult`.

```typescript
import { signMessage } from "@open-wallet-standard/core";

// Sign a UTF-8 message on EVM
const sig = signMessage("agent-treasury", "evm", "hello world");
console.log(sig.signature);  // "0x3a4b5c..." (hex)
console.log(sig.recoveryId); // 0 or 1

// Sign on Solana (Ed25519 — no recoveryId)
const solSig = signMessage("agent-treasury", "solana", "hello world");
console.log(solSig.signature);  // hex
console.log(solSig.recoveryId); // undefined

// Sign hex-encoded binary data with a passphrase
const binSig = signMessage(
  "agent-treasury", "evm", "deadbeef", "my-passphrase", "hex"
);

// Sign using an API key token (policy-gated)
const agentSig = signMessage(
  "agent-treasury", "evm", "hello", "ows_key_a1b2c3d4..."
);

// Sign with a non-default account index
const sig2 = signMessage("agent-treasury", "evm", "hello", undefined, "utf8", 1);
```

---

### signTypedData

Sign EIP-712 typed structured data. EVM only.

```typescript
function signTypedData(
  wallet: string,
  chain: string,
  typedDataJson: string,
  passphrase?: string,
  index?: number,
  vaultPath?: string
): SignResult;
```

| Param | Type | Default | Description |
|-------|------|---------|-------------|
| `wallet` | `string` | — | Wallet name or UUID |
| `chain` | `string` | — | Must be an EVM chain family: `"evm"`, `"base"`, `"ethereum"`, `"polygon"`, etc. |
| `typedDataJson` | `string` | — | JSON string of EIP-712 typed data (domain, types, primaryType, message) |
| `passphrase` | `string` | `undefined` | Decryption passphrase or API key token |
| `index` | `number` | `0` | Account index |
| `vaultPath` | `string` | `"~/.ows"` | Custom vault directory |

**Returns:** `SignResult`.

```typescript
import { signTypedData } from "@open-wallet-standard/core";

const typedData = JSON.stringify({
  types: {
    EIP712Domain: [
      { name: "name", type: "string" },
      { name: "version", type: "string" },
      { name: "chainId", type: "uint256" },
      { name: "verifyingContract", type: "address" },
    ],
    Permit: [
      { name: "owner", type: "address" },
      { name: "spender", type: "address" },
      { name: "value", type: "uint256" },
      { name: "nonce", type: "uint256" },
      { name: "deadline", type: "uint256" },
    ],
  },
  primaryType: "Permit",
  domain: {
    name: "USD Coin",
    version: "2",
    chainId: 8453,
    verifyingContract: "0x833589fCD6eDb6E08f4c7C32D4f71b54bdA02913",
  },
  message: {
    owner: "0x742d35Cc6634C0532925a3b844Bc9e7595f2bD18",
    spender: "0xDEF171Fe48CF0115B1d80b88dc8eAB59176FEe57",
    value: "1000000",
    nonce: "0",
    deadline: "1735689599",
  },
});

const sig = signTypedData("agent-treasury", "evm", typedData, "my-passphrase");
console.log(sig.signature);  // hex-encoded signature
console.log(sig.recoveryId); // 0 or 1
```

---

### signTransaction

Sign a raw transaction without broadcasting.

```typescript
function signTransaction(
  wallet: string,
  chain: string,
  txHex: string,
  passphrase?: string,
  index?: number,
  vaultPath?: string
): SignResult;
```

| Param | Type | Default | Description |
|-------|------|---------|-------------|
| `wallet` | `string` | — | Wallet name or UUID |
| `chain` | `string` | — | Chain family or alias (e.g., `"evm"`, `"base"`, `"solana"`, `"bitcoin"`) |
| `txHex` | `string` | — | Hex-encoded raw transaction bytes |
| `passphrase` | `string` | `undefined` | Decryption passphrase or API key token |
| `index` | `number` | `0` | Account index |
| `vaultPath` | `string` | `"~/.ows"` | Custom vault directory |

**Returns:** `SignResult`.

```typescript
import { signTransaction } from "@open-wallet-standard/core";

// Sign an EVM transaction
const sig = signTransaction(
  "agent-treasury",
  "base",
  "02f87083014a3401843b9aca008504a817c80082520894d8da6bf26964af9d7eed9e03e53415d37aa9604580801ca0",
  "my-passphrase"
);

// Sign a Solana transaction
const solSig = signTransaction("agent-treasury", "solana", "01000103...");

// Sign using an API key token
const agentSig = signTransaction(
  "agent-treasury", "base", "02f8...", "ows_key_a1b2c3d4..."
);
```

---

### signAndSend

Sign a transaction and broadcast it to the network in one call. Requires an RPC endpoint (uses a default public endpoint if not specified).

```typescript
function signAndSend(
  wallet: string,
  chain: string,
  txHex: string,
  passphrase?: string,
  index?: number,
  rpcUrl?: string,
  vaultPath?: string
): SendResult;
```

| Param | Type | Default | Description |
|-------|------|---------|-------------|
| `wallet` | `string` | — | Wallet name or UUID |
| `chain` | `string` | — | Chain family or alias |
| `txHex` | `string` | — | Hex-encoded raw transaction bytes |
| `passphrase` | `string` | `undefined` | Decryption passphrase or API key token |
| `index` | `number` | `0` | Account index |
| `rpcUrl` | `string` | `undefined` | Custom RPC URL. If omitted, a default public endpoint for the chain is used. |
| `vaultPath` | `string` | `"~/.ows"` | Custom vault directory |

**Returns:** `SendResult` with the transaction hash.

```typescript
import { signAndSend } from "@open-wallet-standard/core";

// Sign and broadcast on Base
const result = signAndSend(
  "agent-treasury",
  "base",
  "02f87083014a3401843b9aca008504a817c80082520894d8da6bf26964af9d7eed9e03e53415d37aa9604580801ca0"
);
console.log(result.txHash); // "0xabc123..."

// With a custom RPC endpoint
const result2 = signAndSend(
  "agent-treasury",
  "base",
  "02f8...",
  "my-passphrase",
  0,
  "https://mainnet.base.org"
);

// Using an API key token
const agentResult = signAndSend(
  "agent-treasury",
  "base",
  "02f8...",
  "ows_key_a1b2c3d4..."
);
```

---

## Policy Management

### createPolicy

Register a policy in the vault. Policies are evaluated before every signing operation when an API key is used.

```typescript
function createPolicy(policyJson: string, vaultPath?: string): void;
```

| Param | Type | Default | Description |
|-------|------|---------|-------------|
| `policyJson` | `string` | — | JSON string of the policy definition |
| `vaultPath` | `string` | `"~/.ows"` | Custom vault directory |

The policy JSON must include: `id` (string), `name` (string), `version` (number), `created_at` (ISO 8601), `rules` (array), and `action` (`"deny"`).

**Rule types:**

- `allowed_chains` — `{ "type": "allowed_chains", "chain_ids": ["eip155:8453"] }`
- `expires_at` — `{ "type": "expires_at", "timestamp": "2026-12-31T23:59:59Z" }`

```typescript
import { createPolicy } from "@open-wallet-standard/core";

const policy = {
  id: "base-only-2026",
  name: "Base chain only, expires end of 2026",
  version: 1,
  created_at: "2026-01-01T00:00:00Z",
  rules: [
    { type: "allowed_chains", chain_ids: ["eip155:8453"] },
    { type: "expires_at", timestamp: "2026-12-31T23:59:59Z" },
  ],
  action: "deny",
};

createPolicy(JSON.stringify(policy));
```

---

### listPolicies

List all policies in the vault.

```typescript
function listPolicies(vaultPath?: string): object[];
```

| Param | Type | Default | Description |
|-------|------|---------|-------------|
| `vaultPath` | `string` | `"~/.ows"` | Custom vault directory |

**Returns:** Array of policy objects.

```typescript
import { listPolicies } from "@open-wallet-standard/core";

const policies = listPolicies();
for (const p of policies) {
  console.log(p.id, p.name);
}
```

---

### getPolicy

Retrieve a single policy by ID.

```typescript
function getPolicy(id: string, vaultPath?: string): object;
```

| Param | Type | Default | Description |
|-------|------|---------|-------------|
| `id` | `string` | — | Policy ID |
| `vaultPath` | `string` | `"~/.ows"` | Custom vault directory |

**Returns:** The policy object.

**Throws:** Error if policy not found.

```typescript
import { getPolicy } from "@open-wallet-standard/core";

const policy = getPolicy("base-only-2026");
console.log(policy.rules); // [{ type: "allowed_chains", ... }, ...]
```

---

### deletePolicy

Delete a policy from the vault. Any API keys referencing this policy will no longer be constrained by it.

```typescript
function deletePolicy(id: string, vaultPath?: string): void;
```

| Param | Type | Default | Description |
|-------|------|---------|-------------|
| `id` | `string` | — | Policy ID |
| `vaultPath` | `string` | `"~/.ows"` | Custom vault directory |

```typescript
import { deletePolicy } from "@open-wallet-standard/core";

deletePolicy("base-only-2026");
```

---

## API Key Management

API keys are scoped tokens that let agents sign without ever seeing the private key. Each key is bound to specific wallets and policies. The token (`ows_key_...`) is passed in place of the passphrase — OWS detects the prefix, loads the attached policies, and only signs if every policy allows the operation.

### createApiKey

Create a new API key scoped to specific wallets and policies.

```typescript
function createApiKey(
  name: string,
  walletIds: string[],
  policyIds: string[],
  passphrase: string,
  expiresAt?: string,
  vaultPath?: string
): ApiKeyResult;
```

| Param | Type | Default | Description |
|-------|------|---------|-------------|
| `name` | `string` | — | Human-readable key name |
| `walletIds` | `string[]` | — | Array of wallet IDs (UUID) or names this key can access |
| `policyIds` | `string[]` | — | Array of policy IDs to attach (all must pass for signing to proceed) |
| `passphrase` | `string` | — | The owner passphrase for the wallet(s). Required to re-encrypt the key material for the API token. |
| `expiresAt` | `string` | `undefined` | Expiration timestamp (ISO 8601). If omitted, the key does not expire (rely on policy `expires_at` rules or manual revocation). |
| `vaultPath` | `string` | `"~/.ows"` | Custom vault directory |

**Returns:** `ApiKeyResult`. The `token` field is shown only once — store it securely.

```typescript
import { createApiKey, createPolicy, createWallet } from "@open-wallet-standard/core";

// Set up wallet and policy first
const wallet = createWallet("agent-treasury", "owner-passphrase");

createPolicy(JSON.stringify({
  id: "base-only",
  name: "Base chain only",
  version: 1,
  created_at: new Date().toISOString(),
  rules: [{ type: "allowed_chains", chain_ids: ["eip155:8453"] }],
  action: "deny",
}));

// Create an API key
const key = createApiKey(
  "my-agent-key",
  [wallet.id],
  ["base-only"],
  "owner-passphrase",
  "2026-12-31T23:59:59Z"
);

console.log(key.token); // "ows_key_a1b2c3d4..." — save this!
console.log(key.id);    // UUID for revocation
console.log(key.name);  // "my-agent-key"

// Now the agent can sign using the token
import { signTransaction } from "@open-wallet-standard/core";

const sig = signTransaction("agent-treasury", "base", "02f8...", key.token);
```

---

### listApiKeys

List all API keys in the vault.

```typescript
function listApiKeys(vaultPath?: string): object[];
```

| Param | Type | Default | Description |
|-------|------|---------|-------------|
| `vaultPath` | `string` | `"~/.ows"` | Custom vault directory |

**Returns:** Array of API key metadata objects (tokens are not included — they are shown only at creation time).

```typescript
import { listApiKeys } from "@open-wallet-standard/core";

const keys = listApiKeys();
for (const k of keys) {
  console.log(`${k.name} (${k.id}) — wallets: ${k.walletIds?.join(", ")}`);
}
```

---

### revokeApiKey

Revoke an API key. The token becomes invalid immediately — the re-encrypted key material is deleted from the vault.

```typescript
function revokeApiKey(id: string, vaultPath?: string): void;
```

| Param | Type | Default | Description |
|-------|------|---------|-------------|
| `id` | `string` | — | API key ID (from `createApiKey` result or `listApiKeys`) |
| `vaultPath` | `string` | `"~/.ows"` | Custom vault directory |

```typescript
import { revokeApiKey } from "@open-wallet-standard/core";

revokeApiKey("a1b2c3d4-e5f6-7890-abcd-ef1234567890");
```

---

## Custom Vault Path for Testing

Every function accepts an optional `vaultPath` parameter. Use `mkdtempSync` to create isolated vault directories for tests — this prevents test runs from interfering with each other or with production wallets in `~/.ows/`.

```typescript
import { mkdtempSync, rmSync } from "node:fs";
import { join } from "node:path";
import { tmpdir } from "node:os";
import {
  createWallet,
  listWallets,
  signMessage,
  deleteWallet,
  createPolicy,
  createApiKey,
} from "@open-wallet-standard/core";

// Create an isolated vault for this test
const vaultPath = mkdtempSync(join(tmpdir(), "ows-test-"));

try {
  // All operations use the isolated vault
  const wallet = createWallet("test-wallet", "test-pass", 12, vaultPath);

  const wallets = listWallets(vaultPath);
  console.assert(wallets.length === 1);

  const sig = signMessage(
    "test-wallet", "evm", "test message", "test-pass", "utf8", 0, vaultPath
  );
  console.assert(sig.signature.length > 0);

  // Set up policy + API key in the test vault
  createPolicy(
    JSON.stringify({
      id: "test-policy",
      name: "Test policy",
      version: 1,
      created_at: new Date().toISOString(),
      rules: [{ type: "allowed_chains", chain_ids: ["eip155:8453"] }],
      action: "deny",
    }),
    vaultPath
  );

  const key = createApiKey(
    "test-key",
    [wallet.id],
    ["test-policy"],
    "test-pass",
    undefined,
    vaultPath
  );

  // Sign with the API key token
  const agentSig = signMessage(
    "test-wallet", "evm", "agent message", key.token, "utf8", 0, vaultPath
  );
  console.assert(agentSig.signature.length > 0);

  deleteWallet("test-wallet", vaultPath);
  console.assert(listWallets(vaultPath).length === 0);
} finally {
  // Clean up the temporary vault
  rmSync(vaultPath, { recursive: true, force: true });
}
```

---

## End-to-End Example: Agent Wallet Setup

A complete workflow — create a wallet, set a policy, issue a scoped API key, and sign a transaction as an agent.

```typescript
import {
  createWallet,
  createPolicy,
  createApiKey,
  signTransaction,
  signAndSend,
  revokeApiKey,
  getWallet,
} from "@open-wallet-standard/core";

// 1. Owner creates a wallet
const wallet = createWallet("ops-wallet", "owner-secret", 24);

// 2. Get the Base address for funding
const baseAccount = wallet.accounts.find((a) => a.chainId === "eip155:8453");
console.log(`Fund this address on Base: ${baseAccount.address}`);

// 3. Create a restrictive policy
createPolicy(JSON.stringify({
  id: "agent-policy",
  name: "Base only, expires in 30 days",
  version: 1,
  created_at: new Date().toISOString(),
  rules: [
    { type: "allowed_chains", chain_ids: ["eip155:8453"] },
    {
      type: "expires_at",
      timestamp: new Date(Date.now() + 30 * 24 * 60 * 60 * 1000).toISOString(),
    },
  ],
  action: "deny",
}));

// 4. Issue a scoped API key for the agent
const apiKey = createApiKey(
  "deploy-agent",
  [wallet.id],
  ["agent-policy"],
  "owner-secret"
);
console.log(`Agent token: ${apiKey.token}`);

// 5. Agent signs and broadcasts (agent code — only has the token, never the passphrase)
const result = signAndSend(
  "ops-wallet",
  "base",
  "02f87083014a34...",  // raw tx hex
  apiKey.token            // policy-gated token
);
console.log(`Transaction sent: ${result.txHash}`);

// 6. Revoke when the agent's job is done
revokeApiKey(apiKey.id);
```
