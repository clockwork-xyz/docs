# Queues

**A queue is a** [**Solana account**](https://docs.solana.com/developing/programming-model/accounts) **for managing the state of an on-chain job or workflow.** They are the core Clockwork primitive developers can use to automate their programs.&#x20;

## Account model

Queues track a few basic properties about the workflow they manage such as its owner, triggering condition, kickoff instruction, next instruction, and execution context.&#x20;

```rust
#[account]
#[derive(Debug)]
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

## PDA seeds

```json
[
    SEED_QUEUE,      // The static string "queue"
    queue.authority, // The authority (owner) of the queue
    queue.id         // The id given to this queue by the authority
]
```

## Flow control

<figure><img src="../.gitbook/assets/Blank document (15).png" alt=""><figcaption></figcaption></figure>

<figure><img src="../.gitbook/assets/Blank document (16).png" alt=""><figcaption></figcaption></figure>

## Authorities

On Solana, the term “authority” is used by convention to refer to the owner of a particular asset or account. For Clockwork queues, the authority is its creator (the account which signed the transaction to create the queue). A queue’s authority has the following permissions:

* Pause and resume the queue
* Update the queue’s trigger and first instruction
* Withdraw from the queue’s balance
* Close the queue account

Authorities may be any valid public address (i.e. a wallet pubkey or PDA). When the authority is a PDA, we sometimes refer to this a “program authority” since the authority is an account managed by a program. If you are unfamiliar with PDAs on Solana, the Solana Cookbook has a [great writeup](https://solanacookbook.com/core-concepts/pdas.html) on what they are and how to use them to build secure programs.

## Fees

All queues must maintain a SOL balance to compensate workers for their automation services. Clockwork fees are paid in SOL and charged per crank. The fee is currently set at a flat 1000 lamports per crank and managed by the Clockwork team. This price is subject to change given future price discovery and long term may transition to a more dynamic market-based mechanism.

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
