# Payers

{% embed url="https://github.com/clockwork-xyz/clockwork/blob/781b42fd06f2926899597ce5ea1f19b8ecd3d2e4/utils/src/thread.rs#L13" %}
PAYER\_PUBKEY: Pubkey
{% endembed %}



Anchor currently does not support PDAs as payers for account initialization. This means that if one of your automated instructions initializes a new account, you must specify a keypair signer as the `payer`. For this, we provide a special Clockwork payer address:

```rust
C1ockworkPayer11111111111111111111111111111
```

You can import that address with the [Typescript SDK](https://www.npmjs.com/package/@clockwork-xyz/sdk):

```typescript
import { PAYER_PUBKEY } from "@clockwork-xyz/sdk";
```

You can import that address with the [Rust SDK](https://docs.rs/clockwork-sdk/latest/clockwork\_sdk/utils/static.PAYER\_PUBKEY.html):

```rust
use clockwork_sdk::utils::PAYER_PUBKEY;
```



If an account in your automated instruction references the Clockwork payer account, workers will automatically inject their address in its place. By doing this, the worker node will pay for any account initializations your program needs to do, and Clockwork will reimburse the worker from your thread's account balance.

{% hint style="info" %}
When Anchor adds support for PDA payers (expected in the next release), this “dynamic payer” feature may be deprecated in favor of using PDAs to pay for account initializations.

[https://github.com/coral-xyz/anchor/pull/1938](https://github.com/coral-xyz/anchor/pull/1938)
{% endhint %}
