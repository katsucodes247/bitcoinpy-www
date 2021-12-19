---
title: "P2WSH address (multisig)"
weight: 3
---

{{< tip "warning" >}}
The example for 1-of-1 should only serve as an example. We don't recommend using it in the real
world because it is not its intention. Instead of 1-of-1 use P2PKH!
{{< /tip >}}

## Generate address (1-of-1)

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

# Create a witnessScript. witnessScript in SegWit is equivalent to redeemScript in P2SH transaction,
# however, while the redeemScript of a P2SH transaction is included in the ScriptSig, the 
# WitnessScript is included in the Witness field, making P2WSH inputs cheaper to spend than P2SH 
# inputs.
witness_script = CScript([seckey.pub, OP_CHECKSIG])
script_hash = hashlib.sha256(witness_script).digest()
script_pubkey = CScript([OP_0, script_hash])

# Convert the P2WSH scriptPubKey to a base58 Bitcoin address and print it.
# You'll need to send some funds to it to create a txout to spend.
address = P2WSHBitcoinAddress.from_scriptPubKey(script_pubkey)
print('Address:', str(address))
# outputs: Address: bcrt1qgatzazqjupdalx4v28pxjlys2s3yja9gr3xuca3ugcqpery6c3sqx9g8h7
```

## Spend from address (1-of-1)

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
    script=witness_script,
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
witness = CScriptWitness([sig, witness_script])
tx.wit = CTxWitness([CTxInWitness(witness)])

# Done! Print the transaction
print(b2x(tx.serialize()))
# outputs: 01000000000101c7ceaa5e57552823b8ccd964eb2e94a59f4e7ae722dec521570f20c6862bd7060000000000ffffffff01c09ee60500000000160014a494248aa8067566e8ee2eeb96d553a3140ba6dd02473044022031e858a32ab783a5a1c991b1dc67439d00ac5309ea9c7a144dd742835d9a5abe022030f0142bc37886a0cb73fc7d3701d57b682b12328af0f50bc511089f269e9c010123210378d430274f8c5ec1321338151e9f27f4c676a008bdf8638d07c0b6be9ab35c71ac00000000
```

Now that we have our signed and encoded transaction, we can broadcast it using
[sendrawtransaction](https://chainquery.com/bitcoin-cli/sendrawtransaction):

`bitcoin-cli sendrawtransaction <transaction>`

If the transaction is broadcasted successfully a transaction id will be returned. In this case it was `2f033335f99298cefa1895c399b2ab41f0dd1c9141b746159b2f9416dab53133`.

## Generate address (2-of-2)

In this example we show how to create a 2-of-2 multisig address. This means that two signatures are
required in order to unlock funds.

```py
import hashlib

from bitcoin import SelectParams
from bitcoin.core import b2x, lx, COIN, COutPoint, CMutableTxOut, CMutableTxIn, CMutableTransaction, CTxInWitness, CTxWitness
from bitcoin.core.script import CScript, CScriptWitness, OP_0, OP_2, OP_CHECKMULTISIG, SignatureHash, SIGHASH_ALL, SIGVERSION_WITNESS_V0
from bitcoin.wallet import CBitcoinSecret, CBitcoinAddress, P2WSHBitcoinAddress

# We'll be using regtest throughout this guide
SelectParams("regtest")

# Create the (in)famous correct brainwallet secret key.
# first key
h1 = hashlib.sha256(b'correct horse battery staple first').digest()
seckey1 = CBitcoinSecret.from_secret_bytes(h1)

# second key
h2 = hashlib.sha256(b'correct horse battery staple second').digest()
seckey2 = CBitcoinSecret.from_secret_bytes(h2)

# Create a witnessScript. witnessScript in SegWit is equivalent to redeemScript in P2SH transaction,
# however, while the redeemScript of a P2SH transaction is included in the ScriptSig, the 
# WitnessScript is included in the Witness field, making P2WSH inputs cheaper to spend than P2SH 
# inputs.
witness_script = CScript([OP_2, seckey1.pub, seckey2.pub, OP_2, OP_CHECKMULTISIG])
script_hash = hashlib.sha256(witness_script).digest()
script_pubkey = CScript([OP_0, script_hash])

# Convert the P2WSH scriptPubKey to a base58 Bitcoin address and print it.
# You'll need to send some funds to it to create a txout to spend.
address = P2WSHBitcoinAddress.from_scriptPubKey(script_pubkey)
print('Address:', str(address))
# outputs: Address: bcrt1qljlyqaexx4mmhpl66e6nqdtagjaht87pghuq6p0f98a765c9uj9s3f3ee3
```

## Spend from address (2-of-2)

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

txid = lx("49ff22c9985c1991791b7a3bd6a2e8d1d6567ca283e0885afdc83bd92f56d1c4")
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
destination = CBitcoinAddress("bcrt1q0q579hm06qf655cr6ns274udgf6k9x7nedkeaa").to_scriptPubKey() 
txout = CMutableTxOut(amount_less_fee, destination)

# Create the unsigned transaction.
tx = CMutableTransaction([txin], [txout])

# Calculate the signature hash for that transaction.
sighash = SignatureHash(
    script=witness_script,
    txTo=tx,
    inIdx=0,
    hashtype=SIGHASH_ALL,
    amount=amount,
    sigversion=SIGVERSION_WITNESS_V0,
)

# Now sign it. We have to append the type of signature we want to the end, in this case the usual
# SIGHASH_ALL.
sig1 = seckey1.sign(sighash) + bytes([SIGHASH_ALL])
sig2 = seckey2.sign(sighash) + bytes([SIGHASH_ALL])

# Construct a witness for this P2WSH transaction and add to tx.
witness = CScriptWitness([b"", *[sig1, sig2], witness_script])
tx.wit = CTxWitness([CTxInWitness(witness)])

# Done! Print the transaction
print(b2x(tx.serialize()))
# outputs: 01000000000101c4d1562fd93bc8fd5a88e083a27c56d6d1e8a2d63b7a1b7991195c98c922ff490000000000ffffffff01c09ee605000000001600147829e2df6fd013aa5303d4e0af578d4275629bd30400483045022100fef9da5dfaea90104f033960b00612753197bf96c69fce097699aff60261aa3402203b72ed27c929125e5aa0066e6f641b7ffaee4bb6e4929a2428e79a2e0ed4f0140148304502210094fca9f85165c024cace7e92a099be3199e427103a971fd6336ce7386bb8fe410220046b043475b72ee5f9fa2872344c689ee9c800031010db0c11fdb57b0d37ab6301475221038d19497c3922b807c91b829d6873ae5bfa2ae500f3237100265a302fdce87b052103d3a9dff5a0bb0267f19a9ee1c374901c39045fbe041c1c168d4da4ce0112595552ae00000000
```

Now that we have our signed and encoded transaction, we can broadcast it using
[sendrawtransaction](https://chainquery.com/bitcoin-cli/sendrawtransaction):

`bitcoin-cli sendrawtransaction <transaction>`

If the transaction is broadcasted successfully a transaction id will be returned. In this case it
was `8a5115099d07455670db03f791f3caf9788bb9eea4a0d5b7133c3ff804d262ac`.


## Generate address (1-of-3)

In this example we show how to create a 1-of-3 multisig address. This means that one out of three
signatures can unlock and spend bitcoins.

```py
import hashlib

from bitcoin import SelectParams
from bitcoin.core import b2x, lx, COIN, COutPoint, CMutableTxOut, CMutableTxIn, CMutableTransaction, CTxInWitness, CTxWitness
from bitcoin.core.script import CScript, CScriptWitness, OP_0, OP_1, OP_3, OP_CHECKMULTISIG, SignatureHash, SIGHASH_ALL, SIGVERSION_WITNESS_V0
from bitcoin.wallet import CBitcoinSecret, CBitcoinAddress, P2WSHBitcoinAddress

# We'll be using regtest throughout this guide
SelectParams("regtest")

# Create the (in)famous correct brainwallet secret key.
# first key
h1 = hashlib.sha256(b'correct horse battery staple first').digest()
seckey1 = CBitcoinSecret.from_secret_bytes(h1)

# second key
h2 = hashlib.sha256(b'correct horse battery staple second').digest()
seckey2 = CBitcoinSecret.from_secret_bytes(h2)

# third key
h3 = hashlib.sha256(b'correct horse battery staple third').digest()
seckey3 = CBitcoinSecret.from_secret_bytes(h2)

# Create a witnessScript. witnessScript in SegWit is equivalent to redeemScript in P2SH transaction,
# however, while the redeemScript of a P2SH transaction is included in the ScriptSig, the 
# WitnessScript is included in the Witness field, making P2WSH inputs cheaper to spend than P2SH 
# inputs.
witness_script = CScript([OP_1, seckey1.pub, seckey2.pub, seckey3.pub, OP_3, OP_CHECKMULTISIG])
script_hash = hashlib.sha256(witness_script).digest()
script_pubkey = CScript([OP_0, script_hash])

# Convert the P2WSH scriptPubKey to a base58 Bitcoin address and print it.
# You'll need to send some funds to it to create a txout to spend.
address = P2WSHBitcoinAddress.from_scriptPubKey(script_pubkey)
print('Address:', str(address))
# outputs: Address: bcrt1qt5s78re655w4ls6t2pdgl3mttnqmp3acekgx2n6wuad2cj5cne2s4ktpvj
```

## Spend from address (1-of-3)

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

txid = lx("d8b82af8cf10749dd8bfc703c1ff414ce3d2cdd08c2e2004f720721b91f43fe1")
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
destination = CBitcoinAddress("bcrt1q0q579hm06qf655cr6ns274udgf6k9x7nedkeaa").to_scriptPubKey() 
txout = CMutableTxOut(amount_less_fee, destination)

# Create the unsigned transaction.
tx = CMutableTransaction([txin], [txout])

# Calculate the signature hash for that transaction.
sighash = SignatureHash(
    script=witness_script,
    txTo=tx,
    inIdx=0,
    hashtype=SIGHASH_ALL,
    amount=amount,
    sigversion=SIGVERSION_WITNESS_V0,
)

# Now sign it. We have to append the type of signature we want to the end, in this case the usual
# SIGHASH_ALL.
sig2 = seckey2.sign(sighash) + bytes([SIGHASH_ALL])

# Construct a witness for this P2WSH transaction and add to tx.
witness = CScriptWitness([b"", *[sig2], witness_script])
tx.wit = CTxWitness([CTxInWitness(witness)])

# Done! Print the transaction
print(b2x(tx.serialize()))
# outputs: 01000000000101e13ff4911b7220f704202e8cd0cdd2e34c41ffc103c7bfd89d7410cff82ab8d80000000000ffffffff01c09ee605000000001600147829e2df6fd013aa5303d4e0af578d4275629bd30300483045022100820cf6fc7610c77f5a0df04b58334afe6ee7e70dc000386884049049801c59040220142749c8a88eefc5bf0f123773ac385e79d837b505359c7d87465d843509743101695121038d19497c3922b807c91b829d6873ae5bfa2ae500f3237100265a302fdce87b052103d3a9dff5a0bb0267f19a9ee1c374901c39045fbe041c1c168d4da4ce011259552103d3a9dff5a0bb0267f19a9ee1c374901c39045fbe041c1c168d4da4ce0112595553ae00000000
```

Now that we have our signed and encoded transaction, we can broadcast it using
[sendrawtransaction](https://chainquery.com/bitcoin-cli/sendrawtransaction):

`bitcoin-cli sendrawtransaction <transaction>`

If the transaction is broadcasted successfully a transaction id will be returned. In this case it
was `53e73442d40f8711a5f9b3ea47270498cb3ae8addd9a4d88a1513b48b6aaad19`.

## Generate address (2-of-3)

In this example we show how to create a 2-of-3 multisig address. This means that two out of three
signatures can unlock and spend bitcoins.

```py
import hashlib

from bitcoin import SelectParams
from bitcoin.core import b2x, lx, COIN, COutPoint, CMutableTxOut, CMutableTxIn, CMutableTransaction, CTxInWitness, CTxWitness
from bitcoin.core.script import CScript, CScriptWitness, OP_0, OP_2, OP_3, OP_CHECKMULTISIG, SignatureHash, SIGHASH_ALL, SIGVERSION_WITNESS_V0
from bitcoin.wallet import CBitcoinSecret, CBitcoinAddress, P2WSHBitcoinAddress

# We'll be using regtest throughout this guide
SelectParams("regtest")

# Create the (in)famous correct brainwallet secret key.
# first key
h1 = hashlib.sha256(b'correct horse battery staple first').digest()
seckey1 = CBitcoinSecret.from_secret_bytes(h1)

# second key
h2 = hashlib.sha256(b'correct horse battery staple second').digest()
seckey2 = CBitcoinSecret.from_secret_bytes(h2)

# third key
h3 = hashlib.sha256(b'correct horse battery staple third').digest()
seckey3 = CBitcoinSecret.from_secret_bytes(h2)

# Create a witnessScript. witnessScript in SegWit is equivalent to redeemScript in P2SH transaction,
# however, while the redeemScript of a P2SH transaction is included in the ScriptSig, the 
# WitnessScript is included in the Witness field, making P2WSH inputs cheaper to spend than P2SH 
# inputs.
witness_script = CScript([OP_2, seckey1.pub, seckey2.pub, seckey3.pub, OP_3, OP_CHECKMULTISIG])
script_hash = hashlib.sha256(witness_script).digest()
script_pubkey = CScript([OP_0, script_hash])

# Convert the P2WSH scriptPubKey to a base58 Bitcoin address and print it.
# You'll need to send some funds to it to create a txout to spend.
address = P2WSHBitcoinAddress.from_scriptPubKey(script_pubkey)
print('Address:', str(address))
# outputs: Address: bcrt1qahl75z6ykqwh0sqseqlfvu3z77tzcknwa0pp85f9rrpm33ca8f5qcmgl5s
```

## Spend from address (2-of-3)

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

txid = lx("0acb5f272058c26e5350f1e6f749acdf7233e995f24921d428933817ec46d101")
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
destination = CBitcoinAddress("bcrt1q0q579hm06qf655cr6ns274udgf6k9x7nedkeaa").to_scriptPubKey() 
txout = CMutableTxOut(amount_less_fee, destination)

# Create the unsigned transaction.
tx = CMutableTransaction([txin], [txout])

# Calculate the signature hash for that transaction.
sighash = SignatureHash(
    script=witness_script,
    txTo=tx,
    inIdx=0,
    hashtype=SIGHASH_ALL,
    amount=amount,
    sigversion=SIGVERSION_WITNESS_V0,
)

# Now sign it. We have to append the type of signature we want to the end, in this case the usual
# SIGHASH_ALL.
sig2 = seckey2.sign(sighash) + bytes([SIGHASH_ALL])
sig3 = seckey3.sign(sighash) + bytes([SIGHASH_ALL])

# Construct a witness for this P2WSH transaction and add to tx.
witness = CScriptWitness([b"", *[sig2, sig3], witness_script])
tx.wit = CTxWitness([CTxInWitness(witness)])

# Done! Print the transaction
print(b2x(tx.serialize()))
# outputs: 0100000000010101d146ec17389328d42149f295e93372dfac49f7e6f150536ec25820275fcb0a0000000000ffffffff01c09ee605000000001600147829e2df6fd013aa5303d4e0af578d4275629bd30400483045022100de73bfd2256b962fffea6c6426f3b2d897986ca227ba1e783ed57cc09339bfc102201b91a0811b3e31e06cd19d569c81cd7e17b9b94919b2f0c28b9cad12efa707c301473044022011a7850fa16e11c9e355005b4f4128fe4896790dbe5af6a6aaa12ea009d833cb02205f38839ab79d61b17bd188c89876e1774f947b25ec201c52c3b045863be46d2001695221038d19497c3922b807c91b829d6873ae5bfa2ae500f3237100265a302fdce87b052103d3a9dff5a0bb0267f19a9ee1c374901c39045fbe041c1c168d4da4ce011259552103d3a9dff5a0bb0267f19a9ee1c374901c39045fbe041c1c168d4da4ce0112595553ae00000000
```

Now that we have our signed and encoded transaction, we can broadcast it using
[sendrawtransaction](https://chainquery.com/bitcoin-cli/sendrawtransaction):

`bitcoin-cli sendrawtransaction <transaction>`

If the transaction is broadcasted successfully a transaction id will be returned. In this case it
was `7805449ea7433df7f9674035fdd88052da2084ecf70ca1285efc3e9b4fd0ea44`.
