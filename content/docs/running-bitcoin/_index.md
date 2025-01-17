---
title: "Running Bitcoin"
weight: 2
---

Running Bitcoin is easy. This section with show you how to get your node up and running.

## Installating

We'll be install Bitcoin binary files here. In case you'd prefer to build and install
Bitcoin from source have a look at a relevant `build-` [file](https://github.com/bitcoin/bitcoin/tree/master/doc)
for your operating system.

Below steps are intended for users running Linux. We might add guides for other OS in the
future.

1. Download a relevant file from: https://bitcoincore.org/en/download/
2. Extract the files: `tar xvzf bitcoin-22.0-x86_64-linux-gnu.tar.gz`
3. [Verify your download](https://bitcoincore.org/en/download/#verify-your-download)
4. Copy binary files to local bin directory so they can be accessed from anywhere, eg: `sudo cp bitcoin-22.0/bin/* /usr/local/bin/`

## Configuring

Now that you have successfully installed Bitcoin, we'll go ahead with the configuration.

Bitcoin node can run on several networks:

- `mainnet`: main (production) network, these is the only network where coins have value
- `testnet`: public testnet network with a long history where things are tested, this network
mimics mainnet as much as possible which is why it's based on PoW
- `signet`: new type of public testnet network with centralized consensus where a dedicated
entity or a group with authority to create new blocks can produce new blocks using valid signatures
(not PoW based) 
- `regtest`: private (sandboxed) version of testnet for individual developers where developer
himself has a full control (spinning up and connecting nodes, minig blocks (no actualy PoW needed),
triggering manual reorgs etc.)

During this configuration we will guide you to configure a node that will connect to `testnet`. Later on
we will explain how to connect to other networks. 

#### 1. Create configuration file

- Create a `.bitcoin` folder in your home directory: `mkdir ~/.bitcoin`
- Create a `bitcoin.conf` file to this directory: `touch .bitcoin/bitcoin.conf`

By default all bitcoin binaries read the configuration from `~/.bitcoin/bitcoin.conf` path.

#### 2. Setup configuration file

For more info about specific configuration parameters visit [bitcoin.conf example file](https://github.com/bitcoin/bitcoin/blob/master/share/examples/bitcoin.conf) and [jlopp's config generator](https://jlopp.github.io/bitcoin-core-config-generator/).

```toml
# Generated by https://jlopp.github.io/bitcoin-core-config-generator/

# This config should be placed in following path:
# ~/.bitcoin/bitcoin.conf

# [chain]
# Run this node on the Bitcoin Test Network. Equivalent to -chain=test
testnet=1

# [debug]
# Enable debug logging for all categories.
debug=1

# [network]
# Automatically create Tor hidden service.
listenonion=0

# [rpc]
# Accept command line and JSON-RPC commands.
server=1
rpcuser=user-change-me
rpcpassword=password-change-me


# [Sections]
# Most options automatically apply to mainnet, testnet, and regtest networks.
# If you want to confine an option to just one network, you should add it in the relevant section.
# EXCEPTIONS: The options addnode, connect, port, bind, rpcport, rpcbind and wallet
# only apply to mainnet unless they appear in the appropriate section below.

# Options only for mainnet
[main]

# Options only for testnet
[test]

# Options only for regtest
[regtest]

# Options only for signet
[regtest]
```

#### 3. Run Bitcoin

You can start the node by running `bitcoind`. You should see youre node booting up and starting to
sync. Syncing on testnet takes a few hours and with our configuration it will eat up around 40gb
of disk space.

If you'd prefer to start a `bitcoind` as a background process you need to run it with daemon mode
enabled eg: `bitcoind -daemon`. To see output as you did before you can see the logs in
`.bitcoin/testnet3/debug.log`.

## Congratulations 🎉

You're running Bitcoin.

## Run on other networks

In order to connect to networks other than `testnet` (the one we used in the example config file), you'll need to make some modifications.

#### Mainnet

Modify the config:

1. Under `[chain]` replace `testnet=1` with `mainnet=1`.

```toml
# [chain]
mainnet=1
```

#### Signet

Modify the config:

1. Under `[chain]` replace `testnet=1` with `signet=1`.

```toml
# [chain]
signet=1
```

#### Regtest

Modify the config:

1. Under `[chain]` replace `testnet=1` with `regtest=1`. 

```toml
# [chain]
regtest=1
```

2. Add the following to the `bitcoin.conf` file

```toml
# [relay]
minrelaytxfee=0.00001
```