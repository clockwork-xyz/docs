# Installation

## Rust

Go [**here**](https://www.rust-lang.org/tools/install) to install Rust.

## Solana

Go [**here**](https://docs.solana.com/cli/install-solana-cli-tools) to install Solana, and then run `solana-keygen new` to create a local keypair for development.&#x20;

## Anchor

Go [**here**](https://www.anchor-lang.com/docs/installation) to install Anchor. Anchor is a popular framework for building Solana smart-contracts. We will reference Anchor frequently in these docs and guides.

## Clockwork

To install Clockwork on your local machine, simply run the following cargo install command:

```sh
cargo install -f --locked clockwork-cli
```

If you are on Linux, you may need to additionally install these dependencies:

```sh
sudo apt-get update && \
sudo apt-get upgrade && \
sudo apt-get install -y pkg-config build-essential libudev-dev libssl-dev
```
