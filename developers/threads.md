# Threads

Threads are an automation primitive for Solana developers. Just as a traditional computer program often needs to execute a dynamic series of instructions, so too do Solana programs. Clockwork threads allow developers define a series of instructions (either static or dynamic) and setup a trigger condition to schedule their execution.&#x20;

## Account model

```rust
pub struct Thread {
    /// The owner of this thread.
    pub authority: Pubkey,
    /// The bump, used for PDA validation.
    pub bump: u8,
    /// The cluster clock at the moment the thread was created.
    pub created_at: ClockData,
    /// The context of the thread's current execution state.
    pub exec_context: Option<ExecContext>,
    /// The number of lamports to payout to workers per execution.
    pub fee: u64,
    /// The id of the thread, given by the authority.
    pub id: Vec<u8>,
    /// The instructions to be executed.
    pub instructions: Vec<SerializableInstruction>,
    /// The name of the thread.
    pub name: String,
    /// The next instruction to be executed.
    pub next_instruction: Option<SerializableInstruction>,
    /// Whether or not the thread is currently paused.
    pub paused: bool,
    /// The maximum number of execs allowed per slot.
    pub rate_limit: u64,
    /// The triggering event to kickoff a thread.
    pub trigger: Trigger,

}
```

## Public address

A thread's public address is derived deterministically from its `authority` and `id`. These properties are immutable and may never change throughout the lifetime of the thread account. To verify a Clockwork thread account in your programs, you can use the `pubkey()` helper function provided by the SDK:

```rust
#[account(
    address = thread.pubkey(),
    contraint = thread.id.eq("my_thread"),
    has_one = authority,
)]
pub thread: Account<'info, Thread>
```

## Authority

On Solana, the term “authority” is often used by convention to refer to the user-space owner of a particular account. For Clockwork, a thread's authority is its creator – the account which signed the transaction to originally create the thread. A thread's authority has the following permissions:

* Pause and resume the thread.
* Update the thread’s `trigger` and `instructions`.
* Withdraw from the thread’s balance.
* Close the thread account.

An authority may be any valid public address (i.e. a wallet pubkey or PDA). When the authority is a PDA, we occasionally refer to this as a “program authority” since the account is managed by a program. If you are unfamiliar with PDAs, the Solana Cookbook has a [great writeup](https://solanacookbook.com/core-concepts/pdas.html) on what they are and how to build secure programs with them.

## Fees

Fees are a cost, paid by thread accounts, for the automation services provided by the worker network. The minimum automation fee begins at **0.000001 SOL / instruction.** Thread authorities may choose to pay higher fees to prioritize their threads with worker network.&#x20;

## Dynamic payers

Anchor currently does not support PDAs as payers for account initialization. This means that if one of your automated instructions initializes a new account, you must specify a keypair signer as the `payer`. For this, we provide a special Clockwork payer address:

```rust
C1ockworkPayer11111111111111111111111111111
```

If an account in your automated instruction references the Clockwork payer account, workers will automatically inject their address in its place. By doing this, the worker node will pay for any account initializations your program needs to do, and Clockwork will reimburse the worker from your thread's account balance.

{% hint style="info" %}
When Anchor adds support for PDA payers (expected in the next release), this “payer injection” feature will likely be deprecated in favor of using PDAs to pay for account initializations.

[https://github.com/coral-xyz/anchor/pull/1938](https://github.com/coral-xyz/anchor/pull/1938)
{% endhint %}
