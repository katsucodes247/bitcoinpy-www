---
title: "P2PKH address"
weight: 4
---

P2PKH is an abbreviation for Pay to Public Key Hash.

scriptPubKey: `OP_DUP` `OP_HASH160` `<pubKeyHash>` `OP_EQUALVERIFY` `OP_CHECKSIG`

## Generate address

```py
import hashlib

from bitcoin import SelectParams
from bitcoin.core import b2x, lx, COIN, COutPoint, CMutableTxOut, CMutableTxIn, CMutableTransaction, Hash160
from bitcoin.core.script import CScript, OP_DUP, OP_HASH160, OP_EQUALVERIFY, OP_CHECKSIG, SignatureHash, SIGHASH_ALL
from bitcoin.core.scripteval import VerifyScript, SCRIPT_VERIFY_P2SH
from bitcoin.wallet import CBitcoinAddress, CBitcoinSecret, P2PKHBitcoinAddress

SelectParams('regtest')

# Create the (in)famous correct brainwallet secret key.
h = hashlib.sha256(b'correct horse battery staple').digest()
seckey = CBitcoinSecret.from_secret_bytes(h)

public_key = seckey.pub
address = P2PKHBitcoinAddress.from_pubkey(public_key)
print('Address:', str(address))
# outputs: Address: mrdwvWkma2D6n9mGsbtkazedQQuoksnqJV
```

## Spend from address

Assuming the previously generated address has received funds, we can spend them. In order to spend
them, we'll need information about the transaction id (txid) and a vector of an output (vout). You
can get both from an explorer or by querying your running Bitcoin node by running
[listunspent](https://chainquery.com/bitcoin-cli/listunspent) along with some filters:

`bitcoin-cli listunspent 1 9999999 "[\"address\"]"`

Note that you must have an address in the watchlist in order to get any output. To add an address
to a watchlist run [importaddress](https://chainquery.com/bitcoin-cli/importaddress):

`bitcoin-cli importaddress <address> "<label>" false false`

```py
# we are continuing the code from above

txid = lx("c36a4408a242402cb1584e640a5cbe883513a78aae0cf8ea02986cc76845e9e0")
vout = 0

# Specify the amount send to your P2WSH address.
amount = int(1 * COIN)

# Calculate an amount for the upcoming new UTXO. Set a high fee (5%) to bypass bitcoind minfee
# setting on regtest.
amount_less_fee = amount * 0.99

# Create the txin structure, which includes the outpoint. The scriptSig
# defaults to being empty.
txin = CMutableTxIn(COutPoint(txid, vout))

# We also need the scriptPubKey of the output we're spending because
# SignatureHash() replaces the transaction scriptSig's with it.
#
# Here we'll create that scriptPubKey from scratch using the pubkey that
# corresponds to the secret key we generated above.
scriptPubKey = CScript([OP_DUP, OP_HASH160, Hash160(seckey.pub), OP_EQUALVERIFY, OP_CHECKSIG])

destination = CBitcoinAddress('bcrt1q05cfmjm79ujnnpe2r8wr5kv3kcrtsq3jec3n0l').to_scriptPubKey()

# Create the txout. This time we create the scriptPubKey from a Bitcoin address.
txout = CMutableTxOut(amount_less_fee, destination)

# Create the unsigned transaction.
tx = CMutableTransaction([txin], [txout])

# Calculate the signature hash for that transaction.
sighash = SignatureHash(scriptPubKey, tx, 0, SIGHASH_ALL)

# Now sign it. We have to append the type of signature we want to the end, in
# this case the usual SIGHASH_ALL.
sig = seckey.sign(sighash) + bytes([SIGHASH_ALL])

# Set the scriptSig of our transaction input appropriately.
txin.scriptSig = CScript([sig, seckey.pub])

# Done! Print the transaction
print(b2x(tx.serialize()))
# outputs: 0100000001e0e94568c76c9802eaf80cae8aa7133588be5c0a644e58b12c4042a208446ac3000000006b4830450221009abd104d04ecf518c288979ce387defe2ac1d25cb2cf4b3cd97f54e182ef0c3c022055a558c31bcb79237bb75a6659d07eee7875046693818d6f6f0e6064e733c72401210378d430274f8c5ec1321338151e9f27f4c676a008bdf8638d07c0b6be9ab35c71ffffffff01c09ee605000000001600147d309dcb7e2f2539872a19dc3a5991b606b8023200000000
```

Now that we have our signed and encoded transaction, we can broadcast it using
[sendrawtransaction](https://chainquery.com/bitcoin-cli/sendrawtransaction):

`bitcoin-cli sendrawtransaction <transaction>`

If the transaction is broadcasted successfully a transaction id will be returned. In this case it was `cd6bac96d5f43afaa3663647ed7e7301925ac7760d0d22f1fb7f540a7c080dd7`.

