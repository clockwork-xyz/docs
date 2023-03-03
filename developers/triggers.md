# Triggers

Clockwork provides 5 different trigger conditions for scheduling a thread's execution. These trigger conditions support a wide array of use-cases and allow threads to subscribe to time-based events as well as on-chain state changes. If none of the trigger conditions described below support your team's use-case, please reach out in the [**Clockwork Discord**](localnet.md) to propose a new trigger type.  &#x20;

## 1. Account

Allows a thread to begin execution whenever an account's data changes. This can be useful for listening to account updates, process realtime events, or subscribing to an oracle data stream.

```rust
    Account {
        /// The address of the account to monitor.
        address: Pubkey,
        /// The byte offset of the account data to monitor.
        offset: u64,
        /// The size of the byte slice to monitor (must be less than 1kb)
        size: u64,
    },
```

## 2. Cron

Allows a thread to begin execution whenever a [**cron schedule**](https://en.wikipedia.org/wiki/Cron) is valid. This can be useful for scheduling one-off or periodically recurring actions.

```rust
    Cron {
        /// The schedule in cron syntax. Value must be parsable by the `clockwork_cron` package.
        schedule: String,

        /// Boolean value indicating whether triggering moments may be skipped if they are missed (e.g. due to network downtime).
        /// If false, any "missed" triggering moments will simply be executed as soon as the network comes back online.
        skippable: bool,
    },
```

## 3. Now

Allows a thread to begin execution immediately. This can be useful when a new calculation needs to be immediately kicked off.

```rust
    Now,
```

## 4. Slot

Allows a thread to begin executing when a specified slot has passed. This can be useful for scheduling processes related to staking an Solana epoch transitions.

```rust
    Slot { slot: u64 },
```

## 5. Epoch

Allows a thread to begin executing when a specified epoch becomes active. This can be useful for scheduling processes related to staking and Solana epoch transitions.

```rust
    Epoch { epoch: u64 },
```
