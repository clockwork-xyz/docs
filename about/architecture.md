---
cover: ../.gitbook/assets/gitbook-header-architecture-2 (1).png
coverY: 0
---

# Architecture

## Overview

The Clockwork automation engine has two primary components:

1. A **smart-contract** where users can pay SOL to create transaction queues.
2. A **worker network** where node operators can crank queues and collect automation fees as yield.

These two layers work together to power automations for other programs. At a network level, Clockwork is unique in that every worker is a Solana validator or RPC node under the hood. Since the entire network runs on the same physical servers that already power Solana, Clockwork is generally more reliable, less spammy, and cheaper than traditional SaaS-based alternatives.

<figure><img src="../.gitbook/assets/Blank diagram (5).png" alt=""><figcaption><p>The Clockwork program serves as a proxy-contract which verifies and forwards crank transactions on to your target program.</p></figcaption></figure>

## Transaction queues

As a smart-contract primitive, Clockwork provides **transaction queues** for users to manage the state of on-chain jobs and workflows. Queues tell worker nodes which transactions to execute and pay yield to nodes that submit transactions quickly. They serve as a bi-directional communication and payment channel, allowing users to securely transact with and delegate tasks to the worker network.

<figure><img src="../.gitbook/assets/Blank document (18).png" alt=""><figcaption><p>Queues facilitate a bi-directional exchange of value between users and workers.</p></figcaption></figure>

## Automating signatures&#x20;

**Clockwork is a non-custodial service and does not hold onto users' private keys.** This creates unique security challenges for scheduling transactions on behalf of a user.&#x20;

### Pre-signed transactions

One naive approach to scheduling transactions would be to save pre-signed transaction data somewhere for submission at a later date. This is problematic since it would be impossible to prevent a malicious actor from submitting pre-signed transactions ahead of their intended schedules. Solana explicitly protects against this by requiring every transaction to contain a [**recent blockhash**](https://docs.solana.com/developing/programming-model/transactions#recent-blockhash). This has the consequence of causing Solana transactions to go stale if they're not submitted to blockchain within a couple minutes of being signed.

### Delegated signatories

Instead, Clockwork utilizes a **delegated signatory** model. When a worker submits a crank transaction, the Clockwork program receives it first and adds an additional PDA signer to the transaction context before forwarding it on to the target program.&#x20;

Target programs can verify the Clockwork signature is valid to know if the crank request is safe to process. This proxy-contract model protects programs against spam and from unwanted invocations. Code samples for how to correctly verify crank requests can be found in the [**examples** ](https://github.com/clockwork-xyz/examples/blob/main/hello\_clockwork/programs/hello\_clockwork/src/instructions/hello\_world.rs)repo on Github.&#x20;
