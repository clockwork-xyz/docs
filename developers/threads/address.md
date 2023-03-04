# Address

{% embed url="https://github.com/clockwork-xyz/clockwork/blob/781b42fd06f2926899597ce5ea1f19b8ecd3d2e4/programs/thread/src/state/thread.rs#L38" %}
fn pubkey()
{% endembed %}

A thread's **public address** is derived deterministically from its `authority` and `id`. These properties are immutable and may never change throughout the lifetime of the thread account. To verify a Clockwork thread account in your programs, you can use the `pubkey()` helper function provided by the SDK:

```rust
#[account(
    address = thread.pubkey(),
    contraint = thread.id.eq("my_thread"),
    has_one = authority,
)]
pub thread: Account<'info, Thread>
```
