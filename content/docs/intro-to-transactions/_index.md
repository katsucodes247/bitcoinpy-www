---
title: "Intro to Transactions"
weight: 3
---

Bitcoin doesn't actually have different types of transactions but has different types of addresses
formats. Each address has a different way of "unlocking" its funds. The way that dictates how to
unlock funds is set (or scripted) at the time when the address is generated.

###### P2PKH

Locks bitcoins to the hash of a public key, this is most commonly used for payments.

###### P2SH

Lock bitcoins to the hash of a script, this format enabled more advanced functionalities
(eg. MultiSigs, HTLCs etc.).

###### P2WPKH (SegWit)

Native SegWit address with the same semantics as P2PKH, using this address format makes transactions
cheaper compared to P2PKH .

###### P2WSH (SegWit)

Native SegWit address with the same semantics as P2SH, using this address format makes transactions
cheaper compared to P2PKH.

###### P2TR (Taproot)

Locks bitcoins to a script that can be unlocked by a public key or a [MAST](https://river.com/learn/terms/m/merkelized-alternative-script-tree-mast/) (Merkelized Alternative Script Tree). MAST expands the flexibility and utility of Bitcoin contracts
in an inexpensive and privacy preserving way.

###### P2SH-P2WPKH (Wrapped SegWit)

"Wrapped" SegWit address for P2WPKH it enables software that doesn't support SegWit natively to be
able to send to SegWit addresses. This format is actually P2SH but it uses a native SegWit P2WPKH
script as redeem scripts of a P2SH.


###### P2SH-P2WSH (Wrapped SegWit)

"Wrapped" SegWit address for P2WSH, it enables software that doesn't support SegWit natively to be
able to send to SegWit addresses. This format is actually P2SH but it uses a native SegWit P2WSH
script as redeem scripts  of a P2SH.


------

Sources:

- [Wrapped Segwit](https://river.com/learn/terms/w/wrapped-segwit/)
- [Trezor's Wiki](https://wiki.trezor.io/Glossary)
- [Bitcoin Developer Network](https://bitcoindev.network/)