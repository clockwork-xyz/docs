# List Of Known And Common Issues

## Installation

## CLI
> The CLI crashed with a `FileNotFound` error

You might not have configured a keypair yet, run `solana-keygen` and `solana config set —keypair <FILEPATH>`.


## SDK




## Clockwork Engine
> My validator stopped after ⠚ Initializing..

The Clockwork plugins depends on a specific of Solana _(for know we recommend that you use the exact same version)_.
Make sure your Solana validator uses the same version as Clockwork by installing it again if needed `solana-install init x.y.z`.
- Check the [release notes](https://github.com/clockwork-xyz/clockwork/releases) in doubt.
- If you for some reason you cannot install the same version, please talk to us.
