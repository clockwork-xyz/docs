---
cover: ../.gitbook/assets/gitbook-header-3.png
coverY: 0
---

# Architecture

## Network design

Clockwork has two primary abstraction layers:

1. A **smart-contract** where users can create on-chain task queues.
2. A **worker network** that submits transactions to crank users' task queues.

These two layers work in tandem to power automations for other on-chain programs. What makes Clockwork unique is that all workers on the network are actually Solana validators and RPC nodes under the hood. This design offers significant performance benefits over centralized backends and other automation services. Since the network runs on the same physical hardware as the Solana validators, it is generally cheaper, more reliable, and less spammy than traditional bots.

<figure><img src="../.gitbook/assets/Blank diagram (5).png" alt=""><figcaption><p>The Clockwork program serves as a proxy contract which verifies and forwards cranks on to your target program.</p></figcaption></figure>

## Automating signatures&#x20;

**Clockwork is non-custodial and does not hold onto users' private keys.** This creates unique security challenges for scheduling transactions on behalf of a user.&#x20;

### Pre-signed transactions

One naive approach to scheduling transactions would be to save pre-signed transaction data somewhere for submission at a later date. This is problematic since it would be impossible to prevent malicious actors from submitting the signed transactions ahead of their intended schedule. Solana explicitly protects against this by requiring every transaction to contain a [**recent blockhash**](https://docs.solana.com/developing/programming-model/transactions#recent-blockhash). This has the consequence of causing Solana transactions to go stale if they're not submitted to blockchain within a couple minutes of being signed.

### Delegated signatories

Instead, Clockwork uses a **delegated signatory** model. When a worker builds and submits a crank transaction, the Clockwork program verifies the transaction and adds an additional PDA signer before forwarding the request on to the target program.&#x20;

Target programs can verify the PDA signature from the Clockwork program to know if the crank request is safe to process. This proxy-contract model protects programs against spam and unintended invocations. Code samples showing how correctly verify crank requests can be found in the [**examples** ](https://github.com/clockwork-xyz/examples/blob/main/hello\_clockwork/programs/hello\_clockwork/src/instructions/hello\_world.rs)repo on Github.&#x20;
