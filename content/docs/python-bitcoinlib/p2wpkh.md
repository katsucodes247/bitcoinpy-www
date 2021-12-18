---
title: "P2WPKH address"
weight: 2
---

## Generate address

```py
import hashlib

from bitcoin import SelectParams
from bitcoin.core import b2x, b2lx, lx, COIN, COutPoint, CTxOut, CTxIn, CTxInWitness, CTxWitness, CScriptWitness, CMutableTransaction, Hash160
from bitcoin.core.script import CScript, OP_0, OP_CHECKSIG, SignatureHash, SIGHASH_ALL, SIGVERSION_WITNESS_V0
from bitcoin.wallet import CBitcoinSecret, P2WPKHBitcoinAddress, CBitcoinAddress
from bitcoin.rpc import Proxy

SelectParams("regtest")


# Create the (in)famous correct brainwallet secret key.
h = hashlib.sha256(b'correct horse battery staple').digest()
seckey = CBitcoinSecret.from_secret_bytes(h)

public_key = seckey.pub
script_pubkey = CScript([OP_0, Hash160(public_key)])
address = P2WPKHBitcoinAddress.from_scriptPubKey(script_pubkey)

# Create a witnessScript. witnessScript in SegWit is equivalent to redeemScript in P2PKH
# transaction, however, while the redeemScript of a P2SH transaction is included in the ScriptSig,
# the WitnessScript is included in the Witness field, making P2WPKH inputs cheaper to spend than
# P2PKH inputs.
witness_script = address.to_redeemScript()

print('Address:', str(address))
# outputs: Address: bcrt1q08alc0e5ua69scxhvyma568nvguqccrvah6ml0
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
    script=CScript([seckey.pub, OP_CHECKSIG]),
    txTo=tx,
    inIdx=0,
    hashtype=SIGHASH_ALL,
    amount=amount,
    sigversion=SIGVERSION_WITNESS_V0,
)
signature = seckey.sign(sighash) + bytes([SIGHASH_ALL])

# Construct a witness for this transaction input. The public key is given in
# the witness so that the appropriate redeemScript can be calculated by
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

