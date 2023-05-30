# Support

{% hint style="info" %}
If you have a question, you can ask the community in the [**Clockwork Discord**](https://discord.gg/6zGyWF7mP4). If you would rather have a private word with the team, please file a ticket in the support channel.&#x20;
{% endhint %}

## Why is my thread not executing?

* Is your thread funded account with enough SOL? Check it's balance and airdrop some SOL to it. See the [**Fees**](../about/queues.md#fees) section for more information.
* Is your thread paused? Check its status in the [**Clockwork explorer**](https://app.clockwork.xyz/?cluster=devnet).
* Check for programming errors. Use the [**Clockwork explorer**](https://app.clockwork.xyz/?cluster=devnet) to browse for your thread and click the check if your thread is able to simulate your instructions without errors.&#x20;

#### Common errors

> InstructionDidNotDeserialize and others

Serialization errors can happen when there is a mismatch between your validator's clockwork engine version and the clockwork libraries you are using. Naturally these two need to match:

* Note the version of your `clockwork-client`
* Note the version of your `clockwork-sdk`
* Make sure to checkout and run the same version of the clockwork engine.

> (code -32002) Transaction simulation failed: Attempt to load a program that does not exist

This often happens on localnet. You are probably trying to create a thread by calling the thread program, but that program cannot be found. This is most likely due to running `solana-test-validator` or `anchor localnet` instead of `clockwork localnet`.

> I see in the validator logs: "Expected an executable account"

You created a thread, but that thread is trying to execute an instruction whose program does not exist (hence the not an executable account). You haven't deployed your program yet.

## Why is localnet not working?

> My validator stopped after ⠚ Initializing..

The Clockwork plugins depends on a specific of Solana _(for know we recommend that you use the exact same version)_. Make sure your Solana validator uses the same version as Clockwork by installing it again if needed `solana-install init x.y.z`.

* Check the [**release notes**](https://github.com/clockwork-xyz/clockwork/releases) in doubt.
* If for some reason you cannot install the same version, please talk to us.

> How do I know which version of Solana the Clockwork Engine (geyser plugin) depends on?

Run `cat test-ledger/validator.log | grep "crate-info"`

## How do I write a cron schedule?

The easiest way to test your cron string before even playing with thread, is to install the [**clockwork-cli**](localnet.md#1.-install-the-clockwork-cli) and run `clockwork crontab YOUR_STRING`.

You can use [**https://crontab.guru**](https://crontab.guru/) as reference to build your cron string. Note that crontab guru is a 5 columns cron while the Clockwork [**cron parser**](https://github.com/clockwork-xyz/clockwork/tree/main/cron) is a 7 columns cron. It includes **seconds** (left most column) and **year** (right most column).



