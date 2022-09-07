---
cover: ../.gitbook/assets/gitbook-header-architecture.png
coverY: 0
---

# Architecture

## Network design

Clockwork has two primary abstraction layers:

1. A **smart-contract** where users can create on-chain task queues.
2. A **worker network** which submits transactions to crank users' task queues.

These two layers work together to power any on-chain workflow you can imagine. What makes Clockwork unique is that all workers on the network are actually Solana validators and RPC nodes under the hood. This provides significant performance benefits over centralized backends and other automation services, since the network runs on the same physical hardware as the Solana validators. In general, Clockwork is significantly cheaper, more reliable, and less spammy than traditional bots.

<figure><img src="../.gitbook/assets/Blank diagram (3).png" alt=""><figcaption><p>The Clockwork program serves as a proxy contract which verifies and forwards cranks on to your target program.</p></figcaption></figure>

## Automating signatures&#x20;

Clockwork is a non-custodial service, and thus does not hold onto users' private keys. This creates unique security challenges when it comes to scheduling transactions on behalf users.&#x20;

### Pre-signed transactions

One naive approach to on-chain automation would be to save transaction data with a pre-signed signature. However once a transaction signed, it would impossible able to prevent malicious actors from executing the transaction ahead of its intended schedule. Solana protects against this by requiring that every valid transaction contain a [recent blockhash](https://docs.solana.com/developing/programming-model/transactions#recent-blockhash). This requirement means Solana transactions become stale if they're not submitted within a few minutes of been signed.

### Delegated signatories

Instead, Clockwork uses a **delegated signatory** model. When a worker builds and submits a crank transaction, the Clockwork program verifies the transaction is valid and then adds an additional PDA signer before forwarding the request onto the target program as a cross-program invocation. The target program can verify the additional PDA signer is valid to know if the program invocation is valid and safe to process. This proxy-contract model can protect your program against spam and unwanted callers. Examples of how to verify crank invocations can be found in the [examples](https://github.com/clockwork-xyz/examples/blob/main/hello\_clockwork/programs/hello\_clockwork/src/instructions/hello\_world.rs) repo on Github.&#x20;
