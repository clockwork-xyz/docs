# Dynamic payers

Anchor currently does not support PDAs as payers for account initialization. This means that if one of your instructions initializes a new account, you must specify a keypair signer as the `payer`. For this, we provide a special Clockwork payer address:

```rust
C1ockworkPayer11111111111111111111111111111
```

If an account in your instruction references the Clockwork payer account, workers will automatically inject their address in its place. By doing this, the worker node will pay for any account initializations your programs need to do, and Clockwork will reimburse them from the thread’s balance.

{% hint style="info" %}
Once Anchor adds support for PDA payers (expected in the next release), the “payer injection” feature will be deprecated in favor of using PDAs to pay for account initializations.

[https://github.com/coral-xyz/anchor/pull/1938](https://github.com/coral-xyz/anchor/pull/1938)
{% endhint %}
