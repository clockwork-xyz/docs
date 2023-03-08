# Triggers

{% embed url="https://github.com/clockwork-xyz/clockwork/blob/781b42fd06f2926899597ce5ea1f19b8ecd3d2e4/utils/src/thread.rs#L46" %}
enum Trigger
{% endembed %}

Clockwork provides 5 different **trigger conditions** for scheduling a thread's execution. These trigger conditions support a wide variety of use-cases and allow you to subscribe to time-based as well as event-based changes. If none of the trigger conditions currently provided support your team's use-case, please reach out in the [**Clockwork Discord**](../localnet.md) to propose a new one.

### Account

Allows a thread to begin execution whenever an account's data changes. This can be useful for listening to account updates, process realtime events, or subscribing to an oracle data stream. This trigger type requires 3 arguments.

* `address` – The address of the account to listen to.
* `offset` – The byte offset of the account data to monitor.
* `size` – The size of the byte slice to monitor (must be less than 1kb).

### Cron

Allows a thread to begin execution whenever a [**cron schedule**](https://en.wikipedia.org/wiki/Cron) is valid. This can be useful for scheduling one-off or periodically recurring actions. This trigger type requires 2 arguments.

* `schedule` – A schedule in cron syntax.&#x20;
* `skippable` – Boolean value indicating whether an scheduled invocation can be skipped if the Solana network is unavailable. If false, any missed invocations will be executed as soon as the Solana network is online.&#x20;

### Now

Allows a thread to begin execution immediately. This can be useful when a new calculation needs to be immediately kicked off. This trigger condition takes no arguments.

### Slot

Allows a thread to begin executing when a specified slot has passed. This can be useful for scheduling processes related to staking an Solana epoch transitions. This trigger condition requires 1 argument.

* `slot` – The slot to begin execution on.&#x20;

### Epoch

Allows a thread to begin executing when a specified epoch becomes active. This can be useful for scheduling processes related to staking and Solana epoch transitions. This trigger condition requires 1 argument.&#x20;

* `epoch` – The epoch to begine execution on.&#x20;
