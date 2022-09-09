# Building with queues

**A queue is an** [**account**](https://docs.solana.com/developing/programming-model/accounts) **for managing the state of an on-chain job or workflow.** It is the core developer primitive for building automated programs on Solana.

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

The public address of every queue is derived deterministically from its `authority` and `id`. These properties are immutable and may never change throughout the lifetime of the queue.&#x20;

```json
[
    SEED_QUEUE,      // The static string "queue"
    queue.authority, // The authority (owner) of the queue
    queue.id         // The id given to this queue by the authority
]
```

### Authority

On Solana, the term “authority” is often used by convention to refer to the user-space owner of a particular account. For Clockwork, a queue's authority is its creator – the account which signed the transaction to originally create the queue. A queue’s authority has the following permissions:

* Pause and resume the queue.
* Update the queue’s `trigger` and `first_instruction`.
* Withdraw from the queue’s balance.
* Close the queue account.

An authority may be any valid public address (i.e. a wallet pubkey or PDA). When the authority is a PDA, we occasionally refer to this as a “program authority” since the account is managed by a program. If you are unfamiliar with PDAs, the Solana Cookbook has a [great writeup](https://solanacookbook.com/core-concepts/pdas.html) on what they are and how to build secure programs with them.

### Triggers

Clockwork currently supports 2 trigger types:



1. **Immediate** – Begins executing immediately.&#x20;
   * This trigger can be useful when a process needs to immediately kickoff a complex chain of instructions.
2. **Cron** – Executes according to a [**cron schedule**](https://en.wikipedia.org/wiki/Cron).&#x20;
   * Solana's sysvar clock is used the source-of-truth for time when processing on-chain cron schedules. If Solana's network clock drifts relative to your local wallclock, cron schedules will remain synced to Solana time rather than your local time.
   * If the schedule is recurring and a prior execution context is still active when the triggering moment is met, the queue will finish the prior execution context before kicking off a new one. In other words, queues are single-threaded and should be designed to complete within their schedule's resolution period to avoid drift.
   * If the cron schedule is invalid or has reached a stopping point, the queue will not kickoff a new execution context.

{% hint style="info" %}
New trigger types will be supported soon, including slot-based schedules and event-driven conditions. If you have an idea for a trigger type that is not supported here, please [**file an issue**](https://github.com/clockwork-xyz/clockwork/issues) on Github describing your use-case and ideal interface.
{% endhint %}

## Flow control

As soon as a queue's trigger condition is met, the worker network will begin submitting transactions to "crank" the queue. When this happens, the queue will initialize a new `exec_context` to track its current execution state and send a [**cross-program invocation**](https://docs.solana.com/developing/programming-model/calling-between-programs) to the target program defined by the queue's `kickoff_instruction`.

Here, the target program can do whatever it needs to with the accounts and data it receives. When finished, the program can return a `CrankResponse` and optionally specify a `next_instruction` to be invoked on the next crank of the queue.

<figure><img src="../.gitbook/assets/Blank document (15).png" alt=""><figcaption><p>On the first crank, a queue will execute its <code>kickoff_instruction</code>.</p></figcaption></figure>

Each crank has the responsibility of building the instruction to be invoked on the next crank. In this way, queues provide a simple interface for building complex and dynamically branching workflows via smart-contracts.&#x20;

The worker network will crank a queue indefinitely until either its `next_instruction` value is null, an error is thrown, or the queue's balance is insufficient to pay for the transaction.

<figure><img src="../.gitbook/assets/Blank document (16).png" alt=""><figcaption><p>On subsequent cranks, a queue will execute its <code>next_instruction</code> until a null value is returned.</p></figcaption></figure>

## Payers

Anchor currently does not support PDAs as payers for account initialization. This means that if one of your instructions initializes a new account, you must specify a keypair signer as the `payer`. For this, we provide a special Clocker payer address:

```rust
C1ockworkPayer11111111111111111111111111111
```

If an account in your instruction references the Clockwork payer account, workers will automatically inject their address in its place. By doing this, the worker node will pay for any account initializations your programs need to do, and Clockwork will reimburse them from the queue’s balance.

{% hint style="info" %}
Once Anchor adds support for PDA payers (expected in the next release), the “payer injection” feature will be deprecated in favor of using PDAs to pay for account initializations.

[https://github.com/coral-xyz/anchor/pull/1938](https://github.com/coral-xyz/anchor/pull/1938)
{% endhint %}
