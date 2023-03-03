# Triggers

Clockwork provides three different triggering conditions to kickoff a thread's execution:&#x20;

1. **Account** – Triggers whenever an account's data changes. This can be useful for listening to account updates, process realtime events, or subscribing to an oracle data stream.
2. **Cron** – Triggers according to a [**cron schedule**](https://en.wikipedia.org/wiki/Cron). This can be useful for scheduling one-off or periodically recurring actions.
3. **On Demand** – Begins executing immediately. This trigger type is useful when for immediately kicking off a complex chain of transactions.
