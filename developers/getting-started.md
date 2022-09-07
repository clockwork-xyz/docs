# Getting started

## Build from source

Currently, the fastest way to get started Clockwork is to download the open-source repo, checkout the latest stable version, and build the toolkit from source.

```bash
git clone https://github.com/clockwork-xyz/clockwork
cd clockwork
git checkout tags/$(git describe --tags)
./scripts/build-all.sh . && export PATH=$PWD/bin:$PATH
```

## Deploy to localnet

First, remember to configure your Solana config for local development.

```bash
solana config set --url localhost
```

Launch a Clockwork node for local development.

```bash
clockwork localnet
```

To deploy your program to the Clockwork localnet, you can use either the Anchor or Solana CLIs. As a shortcut, Clockwork also provides a `--bpf-program` flag to deploy the localnet with a pre-built program binary.&#x20;

```shell
# Anchor CLI
anchor deploy

# Solana CLI
solana program deploy --program-id <ADDRESS_OR_KEYPAIR> <PROGRAM_FILEPATH>

# Clockwork CLI
clockwork localnet --bpf-program <ADDRESS_OR_KEYPAIR> <PROGRAM_FILEPATH>
```

## Stream logs

To stream program logs, you can use the Solana CLI.

```bash
solana logs --url localhost
```
