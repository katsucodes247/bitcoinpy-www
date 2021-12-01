---
title: "Transactions"
weight: 2
---

Despite this section being called Transactions, it will actually show how to generate specific types of Bitcoin addresses and then make a transaction to spend the funds that each specific address type will hold.

Bitcoin doesn't actually have transaction types but has different types of addresses. Each type of address has a different way of "unlocking" its funds. The way that dictates how to unlock funds is set (or scripted) at the time when the address is generated.

Bellow are a different types of addresses and way how to spend funds from those addresses. 

## P2WSH

P2WSH is an abbreviation for Pay to Witness Script Hash. P2WSH is the **native Segwit** version of a P2SH.

P2WSH has the same semantics as P2SH, except that the signature is not placed at the same location as before. Segregated Witness (SegWit) moves the proof of ownership from the scriptSig part of the transaction to a new part called the witness of the input. Script Hash allows you to lock coins to the hash of a script, and you then provide that original script when you come unlock those coins.

scriptPubKey: `0` `<witnessScriptHash>`

witnessScriptHash: `sha256(pubKey OP_CHECKSIG)`

### Generate address

```py
import hashlib

from bitcoin import SelectParams
from bitcoin.core import b2x, lx, COIN, COutPoint, CMutableTxOut, CMutableTxIn, CMutableTransaction, CTxInWitness, CTxWitness
from bitcoin.core.script import CScript, CScriptWitness, OP_0, OP_CHECKSIG, SignatureHash, SIGHASH_ALL, SIGVERSION_WITNESS_V0
from bitcoin.wallet import CBitcoinSecret, CBitcoinAddress, P2WSHBitcoinAddress

# We'll be using regtest throughout this guide
SelectParams("regtest")

# Create the (in)famous correct brainwallet secret key.
h = hashlib.sha256(b'correct horse battery staple').digest()
seckey = CBitcoinSecret.from_secret_bytes(h)

# Create a witnessScript and corresponding redeemScript. Similar to a scriptPubKey
# the redeemScript must be satisfied for the funds to be spent.
redeemScript = CScript([seckey.pub, OP_CHECKSIG])
scriptHash = hashlib.sha256(redeemScript).digest()
scriptPubKey = CScript([OP_0, scriptHash])

# Convert the P2WSH scriptPubKey to a base58 Bitcoin address and print it.
# You'll need to send some funds to it to create a txout to spend.
address = P2WSHBitcoinAddress.from_scriptPubKey(scriptPubKey)
print('Address:', str(address))
# outputs: Address: bcrt1qgatzazqjupdalx4v28pxjlys2s3yja9gr3xuca3ugcqpery6c3sqx9g8h7
```

### Spend from address

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

txid = lx("06d72b86c6200f5721c5de22e77a4e9fa5942eeb64d9ccb8232855575eaacec7")
vout = 0

# Specify the amount send to your P2WSH address.
amount = int(1 * COIN)

# Calculate an amount for the upcoming new UTXO. Set a high fee (5%) to bypass bitcoind minfee
# setting on regtest.
amount_less_fee = amount * 0.99

# Create the txin structure, which includes the outpoint. The scriptSig defaults to being empty as
# is necessary for spending a P2WSH output.
txin = CMutableTxIn(COutPoint(txid, vout))

# Specify a destination address and create the txout.
destination = CBitcoinAddress("bcrt1q5j2zfz4gqe6kd68w9m4ed42n5v2qhfkapqadf0").to_scriptPubKey()
txout = CMutableTxOut(amount_less_fee, destination)

# Create the unsigned transaction.
tx = CMutableTransaction([txin], [txout])

# Calculate the signature hash for that transaction.
sighash = SignatureHash(
    script=redeemScript,
    txTo=tx,
    inIdx=0,
    hashtype=SIGHASH_ALL,
    amount=amount,
    sigversion=SIGVERSION_WITNESS_V0,
)

# Now sign it. We have to append the type of signature we want to the end, in this case the usual
# SIGHASH_ALL.
sig = seckey.sign(sighash) + bytes([SIGHASH_ALL])

# Construct a witness for this P2WSH transaction and add to tx.
witness = CScriptWitness([sig, redeemScript])
tx.wit = CTxWitness([CTxInWitness(witness)])

# Done! Print the transaction
print(b2x(tx.serialize()))
# outputs: 01000000000101c7ceaa5e57552823b8ccd964eb2e94a59f4e7ae722dec521570f20c6862bd7060000000000ffffffff01c09ee60500000000160014a494248aa8067566e8ee2eeb96d553a3140ba6dd02473044022031e858a32ab783a5a1c991b1dc67439d00ac5309ea9c7a144dd742835d9a5abe022030f0142bc37886a0cb73fc7d3701d57b682b12328af0f50bc511089f269e9c010123210378d430274f8c5ec1321338151e9f27f4c676a008bdf8638d07c0b6be9ab35c71ac00000000
```

Now that we have our signed and encoded transaction, we can broadcast it using
[sendrawtransaction](https://chainquery.com/bitcoin-cli/sendrawtransaction):

`bitcoin-cli sendrawtransaction <transaction>`

If the transaction is broadcasted successfully a transaction id will be returned. In this case it was `2f033335f99298cefa1895c399b2ab41f0dd1c9141b746159b2f9416dab53133`.

## P2WPKH

P2WPKH is an abbreviation for Pay to Witness Public Key Hash. P2WPKH is the **native Segwit**
version of a P2PKH.

P2WPKH has the same semantics as P2PKH, except that the signature is not placed at the same
location as before. Segregated Witness (SegWit) moves the proof of ownership from the scriptSig
part of the transaction to a new part called the witness of the input. 

scriptPubKey: `0` `<witnessPubKeyHash>`

### Generate address

```py
import hashlib

from bitcoin import SelectParams
from bitcoin.core import b2x, b2lx, lx, COIN, COutPoint, CTxOut, CTxIn, CTxInWitness, CTxWitness, CScriptWitness, CMutableTransaction, Hash160
from bitcoin.core.script import CScript, OP_0, SignatureHash, SIGHASH_ALL, SIGVERSION_WITNESS_V0
from bitcoin.wallet import CBitcoinSecret, P2WPKHBitcoinAddress, CBitcoinAddress
from bitcoin.rpc import Proxy

SelectParams("regtest")


# Create the (in)famous correct brainwallet secret key.
h = hashlib.sha256(b'correct horse battery staple').digest()
seckey = CBitcoinSecret.from_secret_bytes(h)

# Create an address from that private key.
public_key = seckey.pub
scriptPubKey = CScript([OP_0, Hash160(public_key)])
address = P2WPKHBitcoinAddress.from_scriptPubKey(scriptPubKey)
print('Address:', str(address))
# outputs: Address: bcrt1q08alc0e5ua69scxhvyma568nvguqccrvah6ml0
```

### Spend from address

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

txid = lx("f11a1f8c9f1dba811b516e41ddf9159428728d4e98b57dff798619604021b88f")
vout = 1

# Specify the amount send to your P2WSH address.
amount = int(1 * COIN)

# Calculate an amount for the upcoming new UTXO. Set a high fee (5%) to bypass bitcoind minfee
# setting on regtest.
amount_less_fee = amount * 0.99

# Create the txin structure, which includes the outpoint. The scriptSig defaults to being empty as
# is necessary for spending a P2WSH output.
txin = CTxIn(COutPoint(txid, vout))

# Specify a destination address and create the txout.
destination = CBitcoinAddress("bcrt1qvg69hl7uj3y4x3xpy8dq2rrfdhq4nwmzpx9s6y").to_scriptPubKey()

# Create the unsigned transaction.
txin = CTxIn(COutPoint(txid, vout))
txout = CTxOut(amount_less_fee, destination)
tx = CMutableTransaction([txin], [txout])

# Calculate the signature hash for that transaction.
sighash = SignatureHash(
    script=address.to_redeemScript(),
    txTo=tx,
    inIdx=0,
    hashtype=SIGHASH_ALL,
    amount=amount,
    sigversion=SIGVERSION_WITNESS_V0,
)
signature = seckey.sign(sighash) + bytes([SIGHASH_ALL])

# Construct a witness for this transaction input. The public key is given in
# the witness so that the appropriate redeem_script can be calculated by
# anyone. The original scriptPubKey had only the Hash160 hash of the public
# key, not the public key itself, and the redeem script can be entirely
# re-constructed (from implicit template) if given just the public key. So the
# public key is added to the witness. This is P2WPKH in bip141.
witness = [signature, public_key]

# Aggregate all of the witnesses together, and then assign them to the
# transaction object.
ctxinwitnesses = [CTxInWitness(CScriptWitness(witness))]
tx.wit = CTxWitness(ctxinwitnesses)

# Done! Print the transaction
print(b2x(tx.serialize()))
# outputs: 010000000001018fb8214060198679ff7db5984e8d72289415f9dd416e511b81ba1d9f8c1f1af10100000000ffffffff01c09ee6050000000016001462345bffdc94495344c121da050c696dc159bb620247304402200f6ecd46b0b3893a955f29a976fbf2eeb1b9a085ce9873ce678faf04504ed4e902207544896b9f17c2b77ec8bc8416d94b2e2043c7917549e48dfacd1a6f2e6b421b01210378d430274f8c5ec1321338151e9f27f4c676a008bdf8638d07c0b6be9ab35c7100000000
```
Now that we have our signed and encoded transaction, we can broadcast it using
[sendrawtransaction](https://chainquery.com/bitcoin-cli/sendrawtransaction):

`bitcoin-cli sendrawtransaction <transaction>`

If the transaction is broadcasted successfully a transaction id will be returned. In this case it was `210dde7106e38bc8bf761002714e7bd636ee0d5b2dd355327e5dac73b81a68da`.

## P2SH

P2SH is an abbreviation for Pay to Script Hash. It allows you to lock coins to the hash of a
script, and you then provide that original script when you come unlock those coins.

scriptPubKey: `OP_HASH160` `<scriptHash>` `OP_EQUAL`

### Generate address

### Spend from address

## P2PKH

P2PKH is an abbreviation for Pay to Public Key Hash.

scriptPubKey: `OP_DUP` `OP_HASH160` `<pubKeyHash>` `OP_EQUALVERIFY` `OP_CHECKSIG`

### Generate address


### Spend from address

sources:
- https://learnmeabitcoin.com
- https://wiki.trezor.io
- https://bitcoindev.network