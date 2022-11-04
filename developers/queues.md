# Threads

**Transactions threads is are an automation primitive for Solana developers.** Just like traditional applications use threads to execute a series of instructions on a computer, Clockwork provides transaction threads for programs to execute a series of instructions on Solana.&#x20;

## Account model

```rust
pub struct Thread {
    /// The owner of this thread.
    pub authority: Pubkey,
    /// The cluster clock at the moment the thread was created.
    pub created_at: ClockData,
    /// The current execution context.
    pub exec_context: Option<ExecContext>,
    /// The number of lamports to payout to workers per instruction execution.
    pub fee: u64,
    /// The id of the thread, set by the authority.
    pub id: String,
    /// The instruction to kickoff the thread.
    pub kickoff_instruction: InstructionData,
    /// The next instruction in the thread.
    pub next_instruction: Option<InstructionData>,
    /// Whether or not the thread is currently paused.
    pub paused: bool,
    /// The maximum number of instructions allowed per slot.
    pub rate_limit: u64,
    /// The triggering event to kickoff a thread.
    pub trigger: Trigger,
}
```

### Public address

The public address of every thread is derived deterministically from its `authority` and `id`. These properties are immutable and may never change throughout the lifetime of the thread. To verify a Clockwork thread account in your programs, you can use the `pubkey()` helper function provided by the SDK:

```rust
#[account(
    address = thread.pubkey(),
    contraint = thread.id.eq("my_thread"),
    has_one = authority,
)]
pub thread: Account<'info, Thread>
```

### Authority

On Solana, the term “authority” is often used by convention to refer to the user-space owner of a particular account. For Clockwork, a thread's authority is its creator – the account which signed the transaction to originally create the thread. A thread's authority has the following permissions:

* Pause and resume the thread.
* Update the thread’s `trigger` and `kickoff_instruction`.
* Withdraw from the thread’s balance.
* Close the thread account.

An authority may be any valid public address (i.e. a wallet pubkey or PDA). When the authority is a PDA, we occasionally refer to this as a “program authority” since the account is managed by a program. If you are unfamiliar with PDAs, the Solana Cookbook has a [great writeup](https://solanacookbook.com/core-concepts/pdas.html) on what they are and how to build secure programs with them.

### Triggers

Clockwork currently supports 2 trigger types:

1. **Account**
   * Triggers whenever an account's data changes. This trigger type is useful for listening to account updates, process realtime events, or subscribing to an oracle data stream.&#x20;
2. **Cron**&#x20;
   * Triggers according to a [**cron schedule**](https://en.wikipedia.org/wiki/Cron). This trigger type is useful for scheduling one-off or periodically recurring threads.&#x20;
   * Clockwork uses Solana's network clock as the source-of-truth for time when processing cron schedules. If the Solana clock drifts relative to your local wallclock, Clockwork will remain synced to Solana rather than to the time of your local reference frame.
3. **On-demand**  
   * Begins executing immediately. This trigger type is useful when for immediately kicking off a complex chain of transactions.

{% hint style="info" %}
New trigger types will be supported soon. If you have an idea for a trigger type that is not listed here, please [**file an issue**](https://github.com/clockwork-xyz/clockwork/issues) on Github describing your use-case and ideal interface.
{% endhint %}

## Flow control

As soon as a thread's trigger condition is met, the worker network will begin submitting transactions to execute the thread. When this happens, the thread will initialize a new `exec_context` to track its current execution state and send a [**cross-program invocation**](https://docs.solana.com/developing/programming-model/calling-between-programs) to the target program defined by the thread's `kickoff_instruction`.

Here, the target program can do whatever it needs to with the accounts and data it receives. When finished, the program can return a `CrankResponse` and optionally specify a `next_instruction` to be invoked on the next crank of the thread.

<figure><img src="../.gitbook/assets/Blank document (19).png" alt=""><figcaption><p>On the first crank, a thread will execute its <code>kickoff_instruction</code>.</p></figcaption></figure>

Each instruction executed by a thread has the responsibility of building the instruction to be executed. In this way, threads provide a simple interface for building complex and dynamically branching workflows via smart-contracts.&#x20;

The worker network will executing a thread indefinitely until either its `next_instruction` value is null, an error is thrown, or the thread's account balance is insufficient to pay for the transaction.

<figure><img src="../.gitbook/assets/Blank document (20).png" alt=""><figcaption><p>On subsequent cranks, a thread will execute its <code>next_instruction</code> until a null value is returned.</p></figcaption></figure>

## Payers

Anchor currently does not support PDAs as payers for account initialization. This means that if one of your instructions initializes a new account, you must specify a keypair signer as the `payer`. For this, we provide a special Clockwork payer address:

```rust
C1ockworkPayer11111111111111111111111111111
```

If an account in your instruction references the Clockwork payer account, workers will automatically inject their address in its place. By doing this, the worker node will pay for any account initializations your programs need to do, and Clockwork will reimburse them from the thread’s balance.

{% hint style="info" %}
Once Anchor adds support for PDA payers (expected in the next release), the “payer injection” feature will be deprecated in favor of using PDAs to pay for account initializations.

[https://github.com/coral-xyz/anchor/pull/1938](https://github.com/coral-xyz/anchor/pull/1938)
{% endhint %}
