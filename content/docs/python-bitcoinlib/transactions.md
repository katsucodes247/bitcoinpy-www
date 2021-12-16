---
title: "Transactions"
weight: 1
draft: true
---

{{< tip >}}
Library supports mainnet, testnet and regtest. A PR for [signet](https://github.com/petertodd/python-bitcoinlib/pull/266)
has been merged but the version that supports it hasn't been released yet.
{{< /tip >}}

Despite this section being called Transactions, it will actually show how to generate specific
types of Bitcoin addresses and then make a transaction to spend the funds that each specific
address type will hold.

Bitcoin doesn't actually have transaction types but has different types of addresses. Each type of
address has a different way of "unlocking" its funds. The way that dictates how to unlock funds is
set (or scripted) at the time when the address is generated.

Bellow are a different types of addresses and way how to spend funds from those addresses. In the
code samples we will be using **RegTest** network which makes trying out things faster as we can
generate blocks on demand.

<!-- Sources:
- https://learnmeabitcoin.com
- https://wiki.trezor.io
- https://bitcoindev.network
- https://bitcoin.stackexchange.com/a/71456/76798 (help with p2wsh multisig) -->