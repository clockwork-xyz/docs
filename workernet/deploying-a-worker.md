# Deploying a worker

## Download the binary

To turn your Solana validator or RPC into a Clockwork worker, you simply need to install the Clockwork [geyser plugin](https://docs.solana.com/developing/plugins/geyser-plugins). You can get the binary either by [building from source](../developers/localnet.md#build-from-source) or installing the pre-built binary:

```
curl -s https://api.github.com/repos/clockwork-xyz/clockwork/releases/latest | grep "clockwork-geyser-plugin-release-x86_64-unknown-linux-gnu.tar" | cut -d : -f 2,3 | tr -d \" | wget -qi -
tar -xjvf clockwork-geyser-plugin-release-x86_64-unknown-linux-gnu.tar.bz2
rm clockwork-geyser-plugin-release-x86_64-unknown-linux-gnu.tar.bz2
```

## Create a keypair

Next, create a new keypair for signing Clockwork txs. Load this keypair with a small amount of SOL (\~0.01 ◎). You will be compensated for lamports spent by the transactions your worker automates.&#x20;

```
solana-keygen new -o clockwork-worker-keypair.json
```

## Setup the config

Then, setup the plugin config file in a folder where your validator startup script can reference it. Note, the `libpath` and `keypath` values should point to the binary and keypair mentioned in the steps above.

```
{
  "libpath": "/home/sol/clockwork-geyser-plugin-release/lib/libclockwork_plugin.so",
  "keypath": "/home/sol/clockwork-worker-keypair.json",
  "rpc_url": "http://127.0.0.1:8899",
  "slot_timeout_threshold": 150,
  "worker_threads": 10
}
```

## Restart your validator

Finally, add an additional line to your startup script to run your validator with the Clockwork plugin (often located at `/home/sol/bin/validator.sh`):

```
#!/bin/bash

exec solana-validator \
    --identity /home/sol/validator-keypair.json \
    --known-validator dv1ZAGvdsz5hHLwWXsVnM94hWf1pjbKVau1QVkaMJ92 \
    --known-validator dv2eQHeP4RFrJZ6UeiZWoc3XTtmtZCUKxxCApCDcRNV \
    --known-validator dv4ACNkpYPcE3aKmYDqZm9G5EB3J4MRoeE7WNDRBVJB \
    --known-validator dv3qDFk1DTF36Z62bNvrCXe9sKATA6xvVy6A798xxAS \
    --only-known-rpc \
    --full-rpc-api \
    --no-voting \
    --ledger /mnt/ledger \
    --accounts /mnt/accounts \
    --log /home/sol/solana-rpc.log \
    --rpc-port 8899 \
    --rpc-bind-address 0.0.0.0 \
    --dynamic-port-range 8000-8020 \
    --entrypoint entrypoint.devnet.solana.com:8001 \
    --entrypoint entrypoint2.devnet.solana.com:8001 \
    --entrypoint entrypoint3.devnet.solana.com:8001 \
    --entrypoint entrypoint4.devnet.solana.com:8001 \
    --entrypoint entrypoint5.devnet.solana.com:8001 \
    --expected-genesis-hash EtWTRABZaYq6iMfeYKouRu166VU2xqa1wcaWoxPkrZBG \
    --wal-recovery-mode skip_any_corrupted_record \
    --limit-ledger-size \
    
    # Add this line! 👇🏼
    --geyser-plugin-config /home/sol/geyser-plugin-config.json
```

Now simply restart your validator however you normally would!\
