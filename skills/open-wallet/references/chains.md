# Supported Chains Reference

OWS derives accounts for all supported chain families from a single BIP-39 mnemonic. This reference covers identifiers, derivation paths, address formats, and known networks.

---

## Identifier Types (CAIP)

OWS uses [CAIP](https://github.com/ChainAgnostic/CAIPs) identifiers everywhere — wallet files, policy files, audit logs, and SDK calls.

### ChainId (CAIP-2)

Format: `${namespace}:${reference}`

| Example | Meaning |
|---------|---------|
| `eip155:1` | Ethereum mainnet |
| `eip155:8453` | Base |
| `solana:5eykt4UsFv8P8NJdTREpY1vzqKqZKvdp` | Solana mainnet |
| `bip122:000000000019d6689c085ae165831e93` | Bitcoin mainnet |

### AccountId (CAIP-10)

Format: `${ChainId}:${address}`

| Example | Meaning |
|---------|---------|
| `eip155:1:0xab16a96D359eC26a11e2C2b3d8f8B8942d5Bfcdb` | Ethereum account |
| `solana:5eykt4UsFv8P8NJdTREpY1vzqKqZKvdp:7S3P4H...` | Solana account |

### AssetId

Format: `${ChainId}:${contract_address}` or `${ChainId}:native`

| Example | Meaning |
|---------|---------|
| `eip155:8453:native` | Native ETH on Base |
| `eip155:8453:0x833589fCD6eDb6E08f4c7C32D4f71b54bdA02913` | USDC on Base |
| `solana:5eykt4UsFv8P8NJdTREpY1vzqKqZKvdp:native` | Native SOL |

> **IMPORTANT**: Always use CAIP-2 identifiers in code, policy files, and wallet files. Shorthand aliases (like `base` or `ethereum`) only work in CLI context and must never appear in stored data.

---

## Chain Families

| Family | Curve | Coin Type | Derivation Path | Address Format | CAIP-2 Namespace |
|--------|-------|-----------|-----------------|----------------|------------------|
| EVM | secp256k1 | 60 | `m/44'/60'/0'/0/{index}` | EIP-55 checksummed hex (`0x...`) | `eip155` |
| Solana | ed25519 | 501 | `m/44'/501'/{index}'/0'` | Base58 public key | `solana` |
| Bitcoin | secp256k1 | 0 | `m/84'/0'/0'/0/{index}` | Bech32 native segwit (`bc1...`) | `bip122` |
| Cosmos | secp256k1 | 118 | `m/44'/118'/0'/0/{index}` | Bech32 (`cosmos1...`) | `cosmos` |
| Tron | secp256k1 | 195 | `m/44'/195'/0'/0/{index}` | Base58Check (`T...`) | `tron` |
| TON | ed25519 | 607 | `m/44'/607'/{index}'` | Base64url wallet v5r1 (`UQ...`) | `ton` |
| Sui | ed25519 | 784 | `m/44'/784'/{index}'/0'/0'` | `0x` + BLAKE2b-256 hex (32 bytes) | `sui` |
| Spark | secp256k1 | 8797555 | `m/84'/0'/0'/0/{index}` | `spark:` + compressed pubkey hex | `spark` |
| Filecoin | secp256k1 | 461 | `m/44'/461'/0'/0/{index}` | `f1` + base32(blake2b-160) | `fil` |

---

## Known Networks

### EVM Networks

| Name | CAIP-2 Chain ID |
|------|-----------------|
| Ethereum | `eip155:1` |
| Polygon | `eip155:137` |
| Arbitrum | `eip155:42161` |
| Optimism | `eip155:10` |
| Base | `eip155:8453` |
| BSC | `eip155:56` |
| Avalanche | `eip155:43114` |

All EVM networks share the same derivation path, curve, and address format. Only the CAIP-2 reference (chain ID number) differs.

### Non-EVM Networks

| Name | CAIP-2 Chain ID |
|------|-----------------|
| Solana | `solana:5eykt4UsFv8P8NJdTREpY1vzqKqZKvdp` |
| Bitcoin | `bip122:000000000019d6689c085ae165831e93` |
| Cosmos | `cosmos:cosmoshub-4` |
| Tron | `tron:mainnet` |
| TON | `ton:mainnet` |
| Sui | `sui:mainnet` |
| Spark | `spark:mainnet` |
| Filecoin | `fil:mainnet` |

---

## Shorthand Aliases (CLI Only)

These aliases are accepted by the `ows` CLI for convenience. They are resolved to full CAIP-2 identifiers before any processing occurs.

| Shorthand | Full CAIP-2 Identifier |
|-----------|------------------------|
| `ethereum` | `eip155:1` |
| `base` | `eip155:8453` |
| `polygon` | `eip155:137` |
| `arbitrum` | `eip155:42161` |
| `optimism` | `eip155:10` |
| `bsc` | `eip155:56` |
| `avalanche` | `eip155:43114` |
| `solana` | `solana:5eykt4UsFv8P8NJdTREpY1vzqKqZKvdp` |
| `bitcoin` | `bip122:000000000019d6689c085ae165831e93` |
| `cosmos` | `cosmos:cosmoshub-4` |
| `tron` | `tron:mainnet` |
| `ton` | `ton:mainnet` |
| `sui` | `sui:mainnet` |
| `spark` | `spark:mainnet` |
| `filecoin` | `fil:mainnet` |

> **IMPORTANT**: Aliases MUST be resolved to full CAIP-2 identifiers before processing. They MUST NOT appear in wallet files, policy files, or audit logs. If you are writing code or configuration, always use the full CAIP-2 form.

---

## HD Derivation

A single BIP-39 mnemonic derives accounts across all supported chains via hierarchical deterministic (HD) key derivation:

```
Mnemonic (BIP-39)
    |
    v
Master Seed (512 bits via PBKDF2)
    |
    +-- m/44'/60'/0'/0/0    -> EVM Account 0
    +-- m/44'/501'/0'/0'    -> Solana Account 0
    +-- m/84'/0'/0'/0/0     -> Bitcoin Account 0
    +-- m/44'/118'/0'/0/0   -> Cosmos Account 0
    +-- m/44'/195'/0'/0/0   -> Tron Account 0
    +-- m/44'/607'/0'       -> TON Account 0
    +-- m/44'/784'/0'/0'/0' -> Sui Account 0
    +-- m/84'/0'/0'/0/0     -> Spark Account 0
    +-- m/44'/461'/0'/0/0   -> Filecoin Account 0
```

A single mnemonic derives accounts across all supported chains. Each chain family uses its own registered coin type and derivation scheme, so there is no key reuse across families (except Spark and Bitcoin which share the BIP-84 path by design).

To derive additional accounts, increment `{index}` in the derivation path shown in the Chain Families table above.

---

## Adding a New Chain

New chains can be added without changes to OWS core. Chains are added via plugins. A new chain plugin needs:

1. **Canonical CAIP-2 identifier** -- the `namespace:reference` pair for the network.
2. **Derivation path** -- the BIP-44 (or BIP-84) path template with the chain's registered coin type.
3. **Address encoding** -- the algorithm to go from public key to on-chain address.
4. **Signing behavior** -- how transactions and messages are serialized and signed for the chain.

Register the plugin and the chain becomes available across all OWS interfaces (CLI, Node.js SDK, Python SDK, MCP).
