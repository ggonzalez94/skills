---
name: crypto-wallet-for-testing
description: Easy access to a crypto wallet for testing on different chains(it includes bridge instructions if needed). An agent can use these funds freely when he needs to run testnet or mainnet actions that requires funds. It should not be used for actual production use cases
---

# Crypto wallet for testing

A skill that specifies where the PK key, which chains the agent has access to, and the rules for how to move the funds.

## Wallet location

The private key is stored in a simple file under `<PATH>` and can be used freely by the agent.

## How to use it

The wallet has only small amounts, so do not worry too much about security. You can use these funds freely when the user asks you to test things that require eth, usdc or any token on BOTH mainnets and testnets.
If your use case requires spinning up new wallets, you can fund them from this account.

Even if security is not the main concern, avoid:
- Leaking or printing the PK
- Transfering all the funds if it is not necessary

## Bridging

If the user asks you to perform actions where you don't have funds use the `defi` CLI to bridge funds. Prefer LiFi as a provider when possible, since it has support for also bridging funds to chains where you don't have native gas tokens, and get native gas token as well as the bridged token.

If defi CLI is not installed, get it from this repository: https://github.com/ggonzalez94/defi-cli by running: `curl -fsSL https://raw.githubusercontent.com/ggonzalez94/defi-cli/main/scripts/install.sh | sh`