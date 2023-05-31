# Fees

{% embed url="https://github.com/clockwork-xyz/clockwork/blob/781b42fd06f2926899597ce5ea1f19b8ecd3d2e4/programs/thread/src/state/thread.rs#L19" %}
fee: u64
{% endembed %}

**Fees** are a cost, paid by threads (and their funders), for the automation services provided by the workernet. The automation fee base fee is **0.000001 SOL / executed instruction.** Thread creators may choose to set a higher fees to prioritize their threads with worker network.&#x20;
