# Building with queues

**A queue is a** [**Solana account**](https://docs.solana.com/developing/programming-model/accounts) **for managing the state of an on-chain job or workflow.** They are the core primitive developers can use to automate their programs.&#x20;

## Account model

A queue account tracks a few basic properties about the workflow it manages including its owner, triggering condition, kickoff instruction, next instruction, and execution context.&#x20;

```rust
pub struct Queue {
    pub authority: Pubkey,
    pub created_at: ClockData,
    pub exec_context: Option<ExecContext>,
    pub id: String,
    pub is_paused: bool,
    pub kickoff_instruction: InstructionData,
    pub next_instruction: Option<InstructionData>,
    pub trigger: Trigger,
}
```

### Public address

The public address of every queue is derived deterministically from its `authority` and `id` properties. These properties are immutable and may never change.&#x20;

```json
[
    SEED_QUEUE,      // The static string "queue"
    queue.authority, // The authority (owner) of the queue
    queue.id         // The id given to this queue by the authority
]
```

### Authority

On Solana, the term “authority” is used by convention to refer to the owner of a particular asset or account. For Clockwork queues, the authority is its creator – the account which signed the transaction to create the queue. A queue’s authority has the following permissions:

* Pause and resume the queue
* Update the queue’s trigger and first instruction
* Withdraw from the queue’s balance
* Close the queue account

An authority may be any valid public address (i.e. a wallet pubkey or PDA). When the authority is a PDA, we occasionally refer to this as a “program authority” since the account managed by a program. If you are unfamiliar with PDAs, the Solana Cookbook has a [great writeup](https://solanacookbook.com/core-concepts/pdas.html) on what they are and how to use them to build secure programs.

### Triggers

Clockwork currently supports 2 trigger types:

* **Cron** – Executes according to the specified schedule. If the schedule is recurring and a prior execution context is still active when the trigger condition is met, the queue will wait until the prior execution context is finished before kicking off a new one.&#x20;
* **Immediate** – Begins executing immediately and cranks indefinitely until a null `next_instruction` is returned.&#x20;

{% hint style="info" %}
New trigger types will be supported soon including slot-based schedules and event-driven conditions. If you have an idea for a trigger type that is not supported here, please [file an issue](https://github.com/clockwork-xyz/clockwork/issues) on Github describing your use-case and ideal interface.
{% endhint %}

## Flow control

As soon as a queue's trigger condition is met, the worker network will begin submitting transactions to "crank" the queue. When this happens, the queue will initialize a new `exec_context` to track its current execution state and send a CPI to the target program defined in the queue's `kickoff_instruction`.

Here, the target program can do whatever work it needs to with the accounts and data defined in the `kickoff_instruction`. When finished, the target program can return a `CrankResponse` and optionally specify a `next_instruction` to be invoked on the next crank of the queue.

<figure><img src="../.gitbook/assets/Blank document (15).png" alt=""><figcaption><p>On the first crank, a queue will execute its <code>kickoff_instruction</code>.</p></figcaption></figure>

As long as a queue has a non-null `next_instruction` value, the worker network will continue submitting transactions to crank the queue. Each crank has the responsibility of building the instruction to be invoked on the next crank. In this way, queues provide a simple interface for developers to build complex and dynamically branching workflows via smart-contracts.&#x20;

The worker network will automatically crank a queue indefinitely until either its `next_instruction` value is null, an error is thrown, or the queue's balance is insufficient to pay for the transaction.

<figure><img src="../.gitbook/assets/Blank document (16).png" alt=""><figcaption><p>On subsequent cranks, a queue will execute its <code>next_instruction</code> until a null value is returned.</p></figcaption></figure>

## Automation fees

All queues must maintain a sufficient balance of SOL to pay workers for their automation services. Automation fees are paid in [lamports](https://docs.solana.com/introduction#what-are-sols) and charged per crank. The fee is currently set to a flat rate of 1000 lamports per crank and managed by the core Clockwork team. This value is subject to change with future price discovery and long-term may transition to a DAO-controlled or market-based mechanism.

## Payers

Anchor currently does not support PDAs as payers for account initialization. This means that if one of your instructions initializes a new account, you must specify a keypair signer as the `payer`. For this, we provide a special Clocker payer address:

```rust
C1ockworkPayer11111111111111111111111111111
```

If an account in your instruction references the Clockwork payer account, Clockwork will automatically inject the worker’s address in its place when invoking the CPI. By doing this, the worker node will pay for any account initializations and Clockwork will reimburse the worker from your queue’s balance.

{% hint style="info" %}
Once Anchor adds support for PDA payers (expected in the next release), this “payer injection” feature will be deprecated in favor of using PDAs to pay for account initializations.

[https://github.com/coral-xyz/anchor/pull/1938](https://github.com/coral-xyz/anchor/pull/1938)
{% endhint %}
