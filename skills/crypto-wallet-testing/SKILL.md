---
name: crypto-wallet-testing
description: Pre-funded crypto wallet for on-chain testing. Use whenever a task needs ETH, USDC, or other tokens on any EVM chain — deploying contracts, sending transactions, funding sub-wallets, bridging across chains, or running experiments that require real funds. Low-value wallet, not for production.
---

# Crypto Wallet for Testing

A pre-funded wallet you can tap into whenever you need funds for on-chain testing or experiments.
This is a low-value utility wallet — draw what you need, leave the rest for future operations.

You already know how to deploy contracts, send transactions, and interact with chains.
This skill just tells you where the funds are and the rules for using them.

---

## Finding the Wallet

Resolve the private key using this precedence (stop at the first match):

1. **`TESTING_WALLET_PK_FILE` env var** — path to a file containing the hex-encoded private key
   ```bash
   cat "$TESTING_WALLET_PK_FILE"
   ```
2. **Convention path** — `~/.config/crypto-wallet-testing/pk.hex`
   ```bash
   cat ~/.config/crypto-wallet-testing/pk.hex
   ```
3. **`TESTING_WALLET_PK` env var** — raw hex string (less preferred; can leak in shell history/logs)
   ```bash
   echo "$TESTING_WALLET_PK"
   ```

If none are found, stop and tell the user:

> I need a funding wallet for this task. Set one up by either:
> - Creating `~/.config/crypto-wallet-testing/pk.hex` with your hex-encoded private key
> - Setting the `TESTING_WALLET_PK_FILE` env var to point to your key file

### Deriving the Address

Once you have the private key, derive the wallet address using `cast` (from Foundry):

```bash
cast wallet address --private-key "$PK"
```

Check the balance on the target chain before doing anything:

```bash
cast balance "$ADDRESS" --rpc-url <chain_rpc>
```

---

## Rules of Use

### Never print or log the private key
Even for a low-value wallet — if the key leaks into logs, terminal history, or tool output, the wallet can be drained. Load it into variables, pass it to commands, but never echo it or include it in output shown to the user.

### Draw only what the task needs
Don't sweep the entire balance. Future tasks may need these funds too. Transfer the minimum required for the current operation plus a small buffer for gas.

### Prefer funding disposable wallets
For most interactions, create a fresh wallet(or use the one that is being passed in context) and fund it from this one. This limits blast radius — if something goes wrong, only the disposable wallet is exposed, not the funding source + the tests might require a specific address, and not this one.

### Check balance before operating
Before starting any on-chain work:
1. Check the balance on the target chain
2. If insufficient, bridge funds first (see Bridging below)
3. If the funding wallet itself is empty, tell the user

### Gas awareness
Always ensure enough native gas token (ETH) on the target chain(unless using EIP-7702 or ERC-4337 acconts). Bridging tokens without gas means you can't execute transactions. LiFi handles this (see Bridging).

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
