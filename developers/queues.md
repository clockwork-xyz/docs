# Threads

**Threads are an automation primitive for Solana developers.** Just like traditional applications can use threads to execute a series of instructions on a computer, Clockwork provides transaction threads for Solana programs to execute a series of instructions on-chain.

### Triggers

Clockwork currently supports 2 trigger types:

1. **Account**
   * Triggers whenever an account's data changes. This trigger type is useful for listening to account updates, process realtime events, or subscribing to an oracle data stream.
2. **Cron**
   * Triggers according to a [**cron schedule**](https://en.wikipedia.org/wiki/Cron). This trigger type is useful for scheduling one-off or periodically recurring threads.
   * Clockwork uses Solana's network clock as the source-of-truth for time when processing cron schedules. If the Solana clock drifts relative to your local wallclock, Clockwork will remain synced to Solana rather than to the time of your local reference frame.
3. **On-demand** &#x20;
   * Begins executing immediately. This trigger type is useful when for immediately kicking off a complex chain of transactions.

{% hint style="info" %}
New trigger types will be supported soon. If you have an idea for a trigger type that is not listed here, please [**file an issue**](https://github.com/clockwork-xyz/clockwork/issues) on Github describing your use-case and ideal interface.
{% endhint %}

## Flow control

As soon as a thread's trigger condition is met, the worker network will begin submitting transactions to execute the thread. When this happens, the thread will initialize a new `exec_context` to track its current execution state and send a [**cross-program invocation**](https://docs.solana.com/developing/programming-model/calling-between-programs) to the target program defined by the thread's `kickoff_instruction`.

Here, the target program can do whatever it needs to with the accounts and data it receives. When finished, the program can return a `CrankResponse` and optionally specify a `next_instruction` to be invoked on the next crank of the thread.

<figure><img src="../.gitbook/assets/Blank document (19) (1).png" alt=""><figcaption><p>On the first crank, a thread will execute its <code>kickoff_instruction</code>.</p></figcaption></figure>

Each instruction executed by a thread has the responsibility of building the instruction to be executed. In this way, threads provide a simple interface for building complex and dynamically branching workflows via smart-contracts.

The worker network will executing a thread indefinitely until either its `next_instruction` value is null, an error is thrown, or the thread's account balance is insufficient to pay for the transaction.

<figure><img src="../.gitbook/assets/Blank document (20).png" alt=""><figcaption><p>On subsequent cranks, a thread will execute its <code>next_instruction</code> until a null value is returned.</p></figcaption></figure>

## Fees

Fees are a cost, paid by users for automation services provided by the worker network. **** Without fees, the worker network would have no incentive to process automated transactions. The automation fee is currently set to a minimum of **0.000001 SOL / instruction**. Thread owners may set a higher fee on their thread to prioritize their thread with block builders.&#x20;
