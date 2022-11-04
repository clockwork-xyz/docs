# FAQ

## Installation

## CLI
> How do I get information about a thread?

Run `clockwork thread get --address <THREAD_PUBKEY>`

## SDK

## Clockwork Engine
> My validator stopped after ⠚ Initializing..

The Clockwork plugins depends on a specific of Solana _(for know we recommend that you use the exact same version)_.
Make sure your Solana validator uses the same version as Clockwork by installing it again if needed `solana-install init x.y.z`.
- Check the [release notes](https://github.com/clockwork-xyz/clockwork/releases) in doubt.
- If for some reason you cannot install the same version, please talk to us.

> How do I know which version of Solana the Clockwork Engine (geyser plugin) depends on?

Run `cat test-ledger/validator.log | grep "crate-info"`
