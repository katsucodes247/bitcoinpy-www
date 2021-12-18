---
title: "P2SH address (multisig)"
weight: 5
---

{{< tip "warning" >}}
The example for 1-of-1 should only serve as an example. We don't recommend using it in the real
world because it is not its intention. Instead of 1-of-1 use P2PKH!
{{< /tip >}}

### Generate address (1-of-1)

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
redeem_script = CScript([seckey.pub, OP_CHECKSIG])

# Create the magic P2SH scriptPubKey format from that redeemScript. You should
# look at the CScript.to_p2sh_scriptPubKey() function in bitcoin.core.script to
# understand what's happening, as well as read BIP16:
# https://github.com/bitcoin/bips/blob/master/bip-0016.mediawiki
script_pubkey = redeem_script.to_p2sh_scriptPubKey()

# Convert the P2SH scriptPubKey to a base58 Bitcoin address and print it.
# You'll need to send some funds to it to create a txout to spend.
address = CBitcoinAddress.from_scriptPubKey(script_pubkey)
print('Address:',str(address))
# outputs: Address: 2Msc7itHhx2x8MEkTthvtED9pFC36J7QpQb
```

### Spend from address (1-of-1)

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
txout = CMutableTxOut(amount_less_fee, destination)

# Create the unsigned transaction.
tx = CMutableTransaction([txin], [txout])

# Calculate the signature hash for that transaction. Note how the script we use
# is the redeemScript, not the scriptPubKey. That's because when the CHECKSIG
# operation happens EvalScript() will be evaluating the redeemScript, so the
# corresponding SignatureHash() function will use that same script when it
# replaces the scriptSig in the transaction being hashed with the script being
# executed.
sighash = SignatureHash(redeem_script, tx, 0, SIGHASH_ALL)

# Now sign it. We have to append the type of signature we want to the end, in
# this case the usual SIGHASH_ALL.
sig = seckey.sign(sighash) + bytes([SIGHASH_ALL])

# Set the scriptSig of our transaction input appropriately.
txin.scriptSig = CScript([sig, redeem_script])

# Done! Print the transaction
print(b2x(tx.serialize()))
# outputs: 0100000001c56326ed8d353e539fcb1048a76bf604c3f0bf3b935a5b89bf8ba75f13b00b60010000006d483045022100ec13f326674bc6accef9aa8ec7101d1301d8e33af86fc5737c04d66bf6beaa000220095d38069dc8563edfcfb52c36b7b8334d3c76d4ab07fdcd9635bfaf67b19ba60123210378d430274f8c5ec1321338151e9f27f4c676a008bdf8638d07c0b6be9ab35c71acffffffff01c09ee6050000000016001413ab549b66828912f0cbf941f7c0ce0c02e6bc2700000000
```

Now that we have our signed and encoded transaction, we can broadcast it using
[sendrawtransaction](https://chainquery.com/bitcoin-cli/sendrawtransaction):

`bitcoin-cli sendrawtransaction <transaction>`

If the transaction is broadcasted successfully a transaction id will be returned. In this case it
was `5cf00f81103b8b0283d98cec2f20421496eba6cc0660b263275e06f142686650`.


## Generate address (2-of-2)

In this example we show how to create a 2-of-2 multisig address. This means that two signatures are
required in order to unlock funds.

```py
import hashlib

from bitcoin import SelectParams
from bitcoin.core import b2x, lx, COIN, COutPoint, CMutableTxOut, CMutableTxIn, CMutableTransaction, Hash160
from bitcoin.core.script import CScript, OP_DUP, OP_0, OP_2, OP_HASH160, OP_EQUALVERIFY, OP_CHECKMULTISIG, SignatureHash, SIGHASH_ALL
from bitcoin.core.scripteval import VerifyScript, SCRIPT_VERIFY_P2SH
from bitcoin.wallet import CBitcoinAddress, CBitcoinSecret

SelectParams('regtest')

h1 = hashlib.sha256(b'correct horse battery staple first').digest()
seckey1 = CBitcoinSecret.from_secret_bytes(h1)

# second signature
h2 = hashlib.sha256(b'correct horse battery staple second').digest()
seckey2 = CBitcoinSecret.from_secret_bytes(h2)

# Create a redeemScript. Similar to a scriptPubKey the redeemScript must be
# satisfied for the funds to be spent.
redeem_script = CScript([OP_2, seckey1.pub, seckey2.pub, OP_2, OP_CHECKMULTISIG])

# Create the magic P2SH scriptPubKey format from that redeemScript. You should
# look at the CScript.to_p2sh_scriptPubKey() function in bitcoin.core.script to
# understand what's happening, as well as read BIP16:
# https://github.com/bitcoin/bips/blob/master/bip-0016.mediawiki
script_pubkey = redeem_script.to_p2sh_scriptPubKey()

# Convert the P2SH scriptPubKey to a base58 Bitcoin address and print it.
# You'll need to send some funds to it to create a txout to spend.
address = CBitcoinAddress.from_scriptPubKey(script_pubkey)
print('Address:',str(address))
# outputs: Address: 2N3avDJKpr9c8pkRSYgWAsHSVCmRX3ce3w7
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

txid = lx("55a7cdebf307597fade5327daa6c95bcab6abc200a878ec191c1f3bd0b7664d0")
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
    script=redeem_script,
    txTo=tx,
    inIdx=0,
    hashtype=SIGHASH_ALL,
    amount=amount,
)

# Now sign it. We have to append the type of signature we want to the end, in this case the usual
# SIGHASH_ALL.
sig1 = seckey1.sign(sighash) + bytes([SIGHASH_ALL])
sig2 = seckey2.sign(sighash) + bytes([SIGHASH_ALL])

# Construct a witness for this P2WSH transaction and add to tx.
txin.scriptSig = CScript([OP_0, sig1, sig2, redeem_script])

# Done! Print the transaction
print(b2x(tx.serialize()))
# outputs: 0100000001d064760bbdf3c191c18e870a20bc6aabbc956caa7d32e5ad7f5907f3ebcda75500000000db00483045022100a22bf0495398d87538bb07daf62948572def31d411df6048e92a53b00d35f06c02204583f20a0fcbb6f5a978f32dde6dfadfe11581973eef946bb87e1ce5b164fb3a014830450221008611aacb5ab9efb1f64200800ac8f55bb6a1acbcdaa2d3db7741df6b99c9f6f802202be1a1c4fdcb649f622b25ad88befba4ef23689e07409bffd4a1d63237993dbd01475221038d19497c3922b807c91b829d6873ae5bfa2ae500f3237100265a302fdce87b052103d3a9dff5a0bb0267f19a9ee1c374901c39045fbe041c1c168d4da4ce0112595552aeffffffff01c09ee605000000001600147829e2df6fd013aa5303d4e0af578d4275629bd300000000
```

Now that we have our signed and encoded transaction, we can broadcast it using
[sendrawtransaction](https://chainquery.com/bitcoin-cli/sendrawtransaction):

`bitcoin-cli sendrawtransaction <transaction>`

If the transaction is broadcasted successfully a transaction id will be returned. In this case it
was `844dd295da6877b8e7b01fa79f46aedf1b4f21651c0210ebf5e36f2476f2116d`.

