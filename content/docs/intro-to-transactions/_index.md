---
title: "Intro to Transactions"
weight: 3
---

Bitcoin doesn't actually have different types of transactions but has different types of addresses
formats. Each address has a different way of "unlocking" its funds. The way that dictates how to
unlock funds is set (or scripted) at the time when the address is generated.

###### P2PKH

P2PKH is an abbreviation for Pay to Public Key Hash. It locks bitcoins to the hash of a public key,
this is most commonly used for payments.

Note that P2PKH is considered to be a legacy address so P2WPKH should be used instead.


###### P2SH

P2SH is an abbreviation for Pay to Script Hash. It allows you to lock coins to the hash of a
script, and you then provide that original script when you come unlock those coins. P2SH locks
bitcoins to the hash of a script, this format enabled more advanced functionalities (eg. MultiSigs,
HTLCs etc.).

Note that P2SH is considered to be a legacy address so P2WSH should be used instead.

{{< tip >}}
"Script Hash addresses" are intended for multisig or other "smart contract" address. If all
you wish to do is receive payment to an address (without multisig) it's better to use P2WPKH as
it's cheaper to spend from those addresses.
{{< /tip >}}


###### P2WPKH (SegWit)

P2WPKH is an abbreviation for Pay to Witness Public Key Hash. P2WPKH is the **native Segwit** 
version of a P2PKH.

P2WPKH has the same semantics as P2PKH, except that the signature is not placed at the same
location as before. Segregated Witness (SegWit) moves the proof of ownership from the scriptSig
part of the transaction to a new part called the witness of the input which as a result makes
transactions smaller and cheaper.


###### P2WSH (SegWit)

P2WSH is an abbreviation for Pay to Witness Script Hash. P2WSH is the **native Segwit** version of
a P2SH. 

{{< tip >}}
"Script Hash addresses" are intended for multisig or other "smart contract" address. If all
you wish to do is receive payment to an address (without multisig) it's better to use P2WPKH as
it's cheaper to spend from those addresses.
{{< /tip >}}

P2WSH has the same semantics as P2SH, except that the signature is not placed at the same location
as before. Segregated Witness (SegWit) moves the proof of ownership from the scriptSig part of the
transaction to a new part called the witness of the input which as a result makes
transactions smaller and cheaper.

Script Hash allows you to lock coins to the hash of a script, and you then provide that original
script when you come to unlock those coins.


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