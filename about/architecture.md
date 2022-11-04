---
cover: ../.gitbook/assets/gitbook-header-architecture-2 (1).png
coverY: 0
---

# Workernet

{% hint style="info" %}
This page is under construction 👷🏼
{% endhint %}

## Overview

The Clockwork automation engine has two primary subsystem:

1. A **smart-contract** where users can create transaction queues.
2. A **worker network** where Solana node operators can crank queues and collect automation fees as yield.

These two layers work together to power automations for other programs. At a network level, Clockwork is unique in that every worker is a Solana validator or RPC node under the hood. Since the entire worker network runs on the same physical servers that already power the Solana blockchain, Clockwork is generally more reliable, less spammy, and cheaper than traditional SaaS-based alternatives.

<figure><img src="../.gitbook/assets/Blank diagram (3).png" alt=""><figcaption><p>Clockwork straddles the on-chain / off-chain divide.</p></figcaption></figure>

