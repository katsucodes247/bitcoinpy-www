---
title: "Transactions"
weight: 2
---

Despite this section being called Transactions, it will actually show how to generate specific types of Bitcoin addresses and then make a transaction to spend the funds that each specific address type will hold.

Bitcoin doesn't actually have transaction types but has different types of addresses. Each type of address has a different way of "unlocking" its funds. The way that dictates how to unlock funds is set (or scripted) at the time when the address is generated.

Bellow are a different types of addresses and way how to spend funds from those addresses. In the code samples we will be using **RegTest** network which makes trying out things faster as we can generate blocks on demand.

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

```py
import hashlib

from bitcoin import SelectParams
from bitcoin.core import b2x, lx, COIN, COutPoint, CMutableTxOut, CMutableTxIn, CMutableTransaction, Hash160
from bitcoin.core.script import CScript, OP_DUP, OP_HASH160, OP_EQUALVERIFY, OP_CHECKSIG, SignatureHash, SIGHASH_ALL
from bitcoin.core.scripteval import VerifyScript, SCRIPT_VERIFY_P2SH
from bitcoin.wallet import CBitcoinAddress, CBitcoinSecret

SelectParams('regtest')

# Create the (in)famous correct brainwallet secret key.
h = hashlib.sha256(b'correct horse battery staple').digest()
seckey = CBitcoinSecret.from_secret_bytes(h)

# Create a redeemScript. Similar to a scriptPubKey the redeemScript must be
# satisfied for the funds to be spent.
txin_redeemScript = CScript([seckey.pub, OP_CHECKSIG])
print(b2x(txin_redeemScript))

# Create the magic P2SH scriptPubKey format from that redeemScript. You should
# look at the CScript.to_p2sh_scriptPubKey() function in bitcoin.core.script to
# understand what's happening, as well as read BIP16:
# https://github.com/bitcoin/bips/blob/master/bip-0016.mediawiki
txin_scriptPubKey = txin_redeemScript.to_p2sh_scriptPubKey()

# Convert the P2SH scriptPubKey to a base58 Bitcoin address and print it.
# You'll need to send some funds to it to create a txout to spend.
txin_p2sh_address = CBitcoinAddress.from_scriptPubKey(txin_scriptPubKey)
print('Address:',str(txin_p2sh_address))
# outputs: Address: 2Msc7itHhx2x8MEkTthvtED9pFC36J7QpQb
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

txid = lx("600bb0135fa78bbf895b5a933bbff0c304f66ba74810cb9f533e358ded2663c5")
vout = 1

# Specify the amount send to your P2WSH address.
amount = int(1 * COIN)

# Calculate an amount for the upcoming new UTXO. Set a high fee (5%) to bypass bitcoind minfee
# setting on regtest.
amount_less_fee = amount * 0.99

# Create the txin structure, which includes the outpoint. The scriptSig
# defaults to being empty.
txin = CMutableTxIn(COutPoint(txid, vout))

# Specify a destination address and create the txout.
destination = CBitcoinAddress("bcrt1qzw44fxmxs2y39uxtl9ql0sxwpspwd0p8rum3nw").to_scriptPubKey()

# Create the txout. This time we create the scriptPubKey from a Bitcoin address.
txout = CMutableTxOut(amount_less_fee, destination)

# Create the unsigned transaction.
tx = CMutableTransaction([txin], [txout])

# Calculate the signature hash for that transaction. Note how the script we use
# is the redeemScript, not the scriptPubKey. That's because when the CHECKSIG
# operation happens EvalScript() will be evaluating the redeemScript, so the
# corresponding SignatureHash() function will use that same script when it
# replaces the scriptSig in the transaction being hashed with the script being
# executed.
sighash = SignatureHash(txin_redeemScript, tx, 0, SIGHASH_ALL)

# Now sign it. We have to append the type of signature we want to the end, in
# this case the usual SIGHASH_ALL.
sig = seckey.sign(sighash) + bytes([SIGHASH_ALL])

# Set the scriptSig of our transaction input appropriately.
txin.scriptSig = CScript([sig, txin_redeemScript])

# Done! Print the transaction
print(b2x(tx.serialize()))
# outputs: 0100000001c56326ed8d353e539fcb1048a76bf604c3f0bf3b935a5b89bf8ba75f13b00b60010000006d483045022100ec13f326674bc6accef9aa8ec7101d1301d8e33af86fc5737c04d66bf6beaa000220095d38069dc8563edfcfb52c36b7b8334d3c76d4ab07fdcd9635bfaf67b19ba60123210378d430274f8c5ec1321338151e9f27f4c676a008bdf8638d07c0b6be9ab35c71acffffffff01c09ee6050000000016001413ab549b66828912f0cbf941f7c0ce0c02e6bc2700000000
```

Now that we have our signed and encoded transaction, we can broadcast it using
[sendrawtransaction](https://chainquery.com/bitcoin-cli/sendrawtransaction):

`bitcoin-cli sendrawtransaction <transaction>`

If the transaction is broadcasted successfully a transaction id will be returned. In this case it was `5cf00f81103b8b0283d98cec2f20421496eba6cc0660b263275e06f142686650`.

## P2PKH

P2PKH is an abbreviation for Pay to Public Key Hash.

scriptPubKey: `OP_DUP` `OP_HASH160` `<pubKeyHash>` `OP_EQUALVERIFY` `OP_CHECKSIG`

### Generate address

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

address = P2PKHBitcoinAddress.from_pubkey(public_key)
print('Address:', str(address))
# outputs: Address: mrdwvWkma2D6n9mGsbtkazedQQuoksnqJV
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

--------------

Sources:
- https://learnmeabitcoin.com
- https://wiki.trezor.io
- https://bitcoindev.network