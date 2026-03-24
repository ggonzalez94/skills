---
name: crypto-wallet-testing
description: Pre-funded crypto wallet for on-chain testing. Use whenever a task needs ETH, USDC, or other tokens on any EVM chain — deploying contracts, sending transactions, funding operations, bridging across chains, or running experiments that require real funds. Low-value wallet, not for production. Builds on the open-wallet (OWS) skill for wallet management when available. Also handles migration from the old raw-key setup to OWS.
---

# Crypto Wallet for Testing

A pre-funded wallet for on-chain testing and experiments. Use it directly — sign from it, deploy from it, approve from it. No need for intermediary or disposable wallets. This is a low-value utility wallet; OWS keeps the key material safe.

You already know how to deploy contracts, send transactions, and interact with chains.
This skill tells you where the funds are, the rules for using them, and how to migrate from the old setup.

---

## Finding the Wallet

### Step 1: Check for OWS wallet

If OWS is installed (`ows` CLI available), look for a wallet named `testing`:

```bash
ows wallet list
```

If found, use it directly for all operations. See the **open-wallet** skill for full OWS usage.

```bash
# Check balance
ows fund balance --wallet testing --chain base

# Sign a transaction
ows sign tx --wallet testing --chain base --tx "02f8..."

# Sign a message
ows sign message --wallet testing --chain evm --message "hello"
```

**If the OWS wallet exists, you're done — skip to Rules of Use.**

### Step 2: Check for old setup

Look for the old raw-key configuration:

1. `TESTING_WALLET_PK_FILE` env var → file containing hex-encoded private key
2. `~/.config/crypto-wallet-testing/pk.hex` → convention path
3. `TESTING_WALLET_PK` env var → raw hex string

If found AND OWS is installed, **trigger the migration flow** (see Migration below).

If found and OWS is NOT installed, use the raw key with `cast` (Foundry):

```bash
cast wallet address --private-key "$PK"
cast balance "$ADDRESS" --rpc-url <chain_rpc>
```

### Step 3: Nothing found

Tell the user:

> I need a funding wallet for this task. Install OWS and create one:
> ```bash
> curl -fsSL https://docs.openwallet.sh/install.sh | bash
> ows wallet create --name testing
> ```
> Then fund it with some ETH or USDC on your target chain.

---

## Migration from Old Setup to OWS

This flow triggers when the agent detects BOTH an old raw-key setup AND OWS installed, but no OWS wallet named `testing` exists yet.

### Step 1: Import the key into OWS

```bash
cat "$TESTING_WALLET_PK_FILE" | ows wallet import --name testing --private-key
# or if using the convention path:
cat ~/.config/crypto-wallet-testing/pk.hex | ows wallet import --name testing --private-key
```

This creates an OWS wallet with the same key, encrypted in the vault. The old key file is not modified.

### Step 2: Scan for balances

Check balances across the 6 mainnets in chain-reference.md. Use `cast` since the key is already known:

```bash
ADDRESS=$(cast wallet address --private-key "$PK")

# Check native token balance on each mainnet
for chain_rpc in \
  "https://eth.llamarpc.com" \
  "https://rpc.mainnet.taiko.xyz" \
  "https://arb1.arbitrum.io/rpc" \
  "https://mainnet.base.org" \
  "https://mainnet.optimism.io" \
  "https://polygon-rpc.com"; do
  cast balance "$ADDRESS" --rpc-url "$chain_rpc" --ether
done
```

Also check USDC balances using the addresses from `references/chain-reference.md`:

```bash
# Example for Base USDC
cast call 0x833589fCD6eDb6E08f4c7C32D4f71b54bdA02913 \
  "balanceOf(address)(uint256)" "$ADDRESS" \
  --rpc-url "https://mainnet.base.org"
```

### Step 3: Report findings

Tell the user what you found:

> I've imported your old testing wallet into OWS as `testing`.
> Here's what I found across chains:
> - Base: 0.05 ETH, 12.50 USDC
> - Arbitrum: 0.00 ETH
> - (etc.)
>
> Your funds are still in place — the OWS wallet uses the same key, so the addresses are identical for EVM chains. No transfers needed.
>
> Want me to clean up the old key file at `~/.config/crypto-wallet-testing/pk.hex`?

Key point: since OWS imports the same private key, the EVM addresses are identical — there's no need to move funds. The migration is just about getting the key into the encrypted OWS vault.

### Step 4: Clean up (only if user confirms)

```bash
rm ~/.config/crypto-wallet-testing/pk.hex
```

Only delete the old key file if the user explicitly agrees. Never delete it automatically.

---

## Rules of Use

### Use the testing wallet directly

Sign from it, deploy from it, approve contracts from it. There's no need to create disposable or intermediary wallets — this is a low-value testing wallet and OWS keeps the key material encrypted. The simplicity of one wallet outweighs the marginal safety of disposable ones.

### Never print or log the private key

Even for a low-value wallet — if the key leaks into logs, terminal history, or tool output, the wallet can be drained. When using OWS, this is handled automatically — the key never leaves the encrypted vault.

### Draw only what the task needs

Don't sweep the entire balance. Future tasks may need these funds too. Transfer the minimum required for the current operation plus a small buffer for gas.

### Check balance before operating

Before starting any on-chain work:
1. Check the balance on the target chain
2. If insufficient, bridge funds first (see Bridging below)
3. If the funding wallet itself is empty, tell the user

### Gas awareness

Always ensure enough native gas token (ETH) on the target chain (unless using EIP-7702 or ERC-4337 accounts). Bridging tokens without gas means you can't execute transactions. LiFi handles this (see Bridging).

---

## Bridging

When the target chain doesn't have enough funds, bridge from a chain where the wallet has a balance.

### Tool: `defi` CLI

The preferred bridging tool. If not installed:

```bash
curl -fsSL https://raw.githubusercontent.com/ggonzalez94/defi-cli/main/scripts/install.sh | sh
```

### Provider: LiFi (preferred)

Use LiFi as the bridge provider when possible. Key advantage: it can deliver **both native gas and the bridged token** to chains where the wallet has zero balance. This solves the cold-start problem on new chains.

### Bridging workflow

1. Identify which chain has funds (check balances across major chains)
2. Bridge to the target chain using `defi` CLI with LiFi
3. Request native gas as part of the bridge if the target chain has zero ETH
4. Verify arrival before proceeding with the original task

---

## Quick Reference

See [`references/chain-reference.md`](references/chain-reference.md) for:
- Chain IDs (don't guess these — they differ per network)
- Common token addresses per chain (USDC, USDT, WETH — these vary across chains)

For full wallet management, multi-chain support, policies, and agent access, see the **open-wallet** skill.
