# FAQ

## Why is my thread not executing?

* Check for programming errors. Use the [Clockwork explorer](https://explorer.clockwork.xyz) to browse for your thread and click the **Simulate thread** button to see if your thread is able to simulate your instructions without errors.&#x20;
* Is your thread funded account with enough SOL? Check it's balance and airdrop some SOL to it. See the [Fees](../about/queues.md#fees) section for more information.
* Is your thread paused? Check its status in the [Clockwork explorer](https://explorer.clockwork.xyz/).

## Common programming bugs

> InstructionDidNotDeserialize and others

Serialization errors can happen when there is a mismatch between your validator's clockwork engine version and the clockwork libraries you are using. Naturally these two need to match:

* Note the version of your `clockwork-client`
* Note the version of your `clockwork-sdk`
* Make sure to checkout and run the same version of the clockwork engine.

> (code -32002) Transaction simulation failed: Attempt to load a program that does not exist

This often happens on localnet. You are probably trying to create a thread by calling the thread program, but that program cannot be found. This is most likely due to running `solana-test-validator` or `anchor localnet` instead of `clockwork localnet`.

> I see in the validator logs: "Expected an executable account"

You created a thread, but that thread is trying to execute an instruction whose program does not exist (hence the not an executable account). You haven't deployed your program yet.

## Localnet issues

> My validator stopped after ⠚ Initializing..

The Clockwork plugins depends on a specific of Solana _(for know we recommend that you use the exact same version)_. Make sure your Solana validator uses the same version as Clockwork by installing it again if needed `solana-install init x.y.z`.

* Check the [release notes](https://github.com/clockwork-xyz/clockwork/releases) in doubt.
* If for some reason you cannot install the same version, please talk to us.

> How do I know which version of Solana the Clockwork Engine (geyser plugin) depends on?

Run `cat test-ledger/validator.log | grep "crate-info"`

## Debugging cron schedules

Not sure if you have a valid cron expression? You can use [https://crontab.guru](https://crontab.guru/) to check your cron string. Note that crontab guru is a 5 columns cron while The [clockwork cron parser](https://github.com/clockwork-xyz/clockwork/tree/main/cron) is a 7 columns cron. It includes **seconds** (left most column) and **year** (right most column).

## Looking up a parsed thread account

Use the Clockwork CLI to run the following command:

```
 clockwork thread get --address <THREAD_PUBKEY>
```

