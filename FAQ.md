# FAQ

## Installation
TBC

## Common Errors
> (code -32002) Transaction simulation failed: Attempt to load a program that does not exist

Often happens on localnet, most probably you are trying to create a thread by calling the thread progrma, but that program cannot be found. That's probably because you ran `solana-test-validator` instead of `clockwork localnet`.

> I see in the validator logs: "Expected an executable account"

You created a thread, but that thread is looking to execute an instruction whose program does not exist (hence the not an executable account). You haven't deployed your program yet.


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
