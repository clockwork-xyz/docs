# FAQ

## Installation
TBC

## Why Is My Thread Not Executing?
- **Instruction Error:** Use the explorer to browse for your Thread and click the simulate **Thread Button** to see if the Thread is able to simulate your Instruction without errors, here's an [example thread](https://explorer.clockwork.xyz/address/9e3BzA7F9CHTYHyTcZ1pja8dY3c9cjLmUPiuY5bpuguX?network=devnet)
- **Fees:** Does your thread account have enough SOL? Check the economics https://docs.clockwork.xyz/about/queues#fees
- **Status:** Is your thread paused? Check the status in the [Clockwork Explorer](https://explorer.clockwork.xyz)

## Common Errors
> Localnet - InstructionDidNotDeserialize and others

Serialization errors happen when there is a mismatch between your validator's clockwork engine version and the clockwork librairies you are using. Naturally these two need to match:
- Note the version of your `clockwork-client`
- Note the version of your `clockwork-sdk`
- Make sure to checkout and run the same version of the clockwork engine

> (code -32002) Transaction simulation failed: Attempt to load a program that does not exist

Often happens on localnet, most probably you are trying to create a thread by calling the thread progrma, but that program cannot be found. That's probably because you ran `solana-test-validator` instead of `clockwork localnet`.

> I see in the validator logs: "Expected an executable account"

You created a thread, but that thread is looking to execute an instruction whose program does not exist (hence the not an executable account). You haven't deployed your program yet.

## Triggers
### Cron
How to write the cron expression? You can use https://crontab.guru to check your cron string.
The [clockwork cron parser](https://github.com/clockwork-xyz/clockwork/tree/main/cron) however includes **seconds** (left most column) and **year** (right most column).

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
