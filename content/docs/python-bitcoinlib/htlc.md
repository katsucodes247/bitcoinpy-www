---
title: "HTLC"
weight: 6
---

Below is a **sample** Hash Time Locked Contract (HTLC). It's a conditional payment that can be spent
in two ways. Either sender (the person who locked the funds in) can unlock them after some number of
blocks have been mined or the receiver (the person whom the funds are intended for) when he gets
reveald the secret code that only sender knows.

Note that each condition also runs a `OP_EQUALVERIFY` check on a public key to enforce that:
- only the sender can spend after X amount of blocks are mined
- only the recipient can spend if he knows the secret (sender or anyone else can not spend these coins if they know the secret)

HTLC are used in Lightning Network, atomic swaps, same-chain coin swaps and other advanced protocols.

There are different ways of how HTLC's can be constructed and bellow example is just one of the way and not
necesseraly the best way. For example `OP_CHECKLOCKTIMEVERIFY` would better ber replaced with `OP_CSV`, and
the script itself could be optimized to be smaller in size.

## Generate address

```py
import hashlib

from bitcoin import SelectParams
from bitcoin.core import b2x, x, lx, COIN, COutPoint, CMutableTxOut, CMutableTxIn, CMutableTransaction, Hash160, CScriptWitness, CTxInWitness, CTxWitness
from bitcoin.core.script import CScript, OP_0, OP_IF, OP_ELSE, OP_SHA256, OP_DUP, OP_HASH160, OP_EQUALVERIFY, OP_CHECKLOCKTIMEVERIFY, OP_CHECKSIG, SignatureHash, SIGHASH_ALL, OP_DROP, OP_ENDIF, SIGVERSION_WITNESS_V0
from bitcoin.wallet import CBitcoinAddress, CBitcoinSecret, P2WSHBitcoinAddress

SelectParams('regtest')

# addresses generate via bitcoin-cli public key and private key were fetched using bitcoin-cli commands: `getaddressinfo` and `dumpprivkey`
# 1. (sender): bcrt1qz52gzlzcesun0cy4v8u5k6uwrjmqayhvf0w806 (03d28046cd12e83832cca3fc4428d254e60092e06fa3f3b8de32062c1b07f58976)
# 2. (recipient): bcrt1q3qk7a5d2963feda6d6lrrvrvsnp05zlzasme8f (02743d0627a342afdcd7b1577d4d175f0c5206bc3da193bfecd5594b2eba1d256c)

# private key for recipient bcrt1q3qk7a5d2963feda6d6lrrvrvsnp05zlzasme8f (generated via bitcoin-cli)
private_key_recipient = "cR6XDAkF7urxoX3PioYfWwCB5MJTVuCNTdeUc5t2nceKPL2nBmfJ"
seckey_recipient = CBitcoinSecret(private_key_recipient)

# private key for sender bcrt1qz52gzlzcesun0cy4v8u5k6uwrjmqayhvf0w806 (generated via bitcoin-cli)
private_key_sender = "cNQREEPKSd7ugdCMAQNEedNpYhGrgXiqfyWSt4H7jfm722FWFz2V"
seckey_sender = CBitcoinSecret(private_key_sender)

# secret and preimage
secret = b"super secret code"
preimage = hashlib.sha256(secret).digest()

lockduration = 5  # in 5 blocks
current_blocknum = 274  # current number of blocks
redeem_blocknum = current_blocknum + lockduration
recipientpubkey = x("02743d0627a342afdcd7b1577d4d175f0c5206bc3da193bfecd5594b2eba1d256c")
senderpubkey = x("03d28046cd12e83832cca3fc4428d254e60092e06fa3f3b8de32062c1b07f58976")

# Create a htlc script where in order to spend either of the two conditions need to be met:
# 1. recipient can claim funds by providing a secret
# 2. sender can claim funds after X blocks have been mined
witness_script = CScript([
    OP_IF,
        OP_SHA256, preimage, OP_EQUALVERIFY, OP_DUP, OP_HASH160, Hash160(recipientpubkey),
    OP_ELSE,
        redeem_blocknum, OP_CHECKLOCKTIMEVERIFY, OP_DROP, OP_DUP, OP_HASH160, Hash160(senderpubkey),
    OP_ENDIF,
    OP_EQUALVERIFY,
    OP_CHECKSIG,
])

script_hash = hashlib.sha256(witness_script).digest()
script_pubkey = CScript([OP_0, script_hash])
address = P2WSHBitcoinAddress.from_scriptPubKey(script_pubkey)
print('Address:', str(address))
# Address: bcrt1q5xpn8y8hlkf5nqc2sqkph64qmj0exf6cwtlqknf34y9s4ejga2zs3f2m9s
```

## Spend from address (via secret)

In this example we will spend the funds by using the secret.

```py
# we are continuing the code from above

txid = lx("2c3759145da44e8a75d24e3535f643de4e28fb85e9f779fa1d0255c7f8b17729")
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
destination = CBitcoinAddress("bcrt1q3qk7a5d2963feda6d6lrrvrvsnp05zlzasme8f").to_scriptPubKey() 
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
sig = seckey_recipient.sign(sighash) + bytes([SIGHASH_ALL])

# Construct a witness for this P2WSH transaction and add to tx.
witness = CScriptWitness([sig, seckey_recipient.pub, secret, b'\x01', witness_script])
tx.wit = CTxWitness([CTxInWitness(witness)])

# Done! Print the transaction
print(b2x(tx.serialize()))
# outputs: 010000000001012977b1f8c755021dfa79f7e985fb284ede43f635354ed2758a4ea45d1459372c0000000000ffffffff01c09ee60500000000160014882deed1aa2ea29cb7ba6ebe31b06c84c2fa0be205473044022023b1482fb9e119eaac4c623ded42aece99b468d39f6a0a2a21bdb0320a9f2a770220237b99c6063061ae72dd4b39b9fb6390a3f070cdb2b7dee936a8cb0f359cdb7f012102743d0627a342afdcd7b1577d4d175f0c5206bc3da193bfecd5594b2eba1d256c1173757065722073656372657420636f646501015b63a8206bd66227651d0fe5c43863d7b29a4097e31dbf51e6603eee75947bff5e96ae438876a914882deed1aa2ea29cb7ba6ebe31b06c84c2fa0be267021701b17576a9141514817c58cc3937e09561f94b6b8e1cb60e92ec6888ac00000000
```

Now that we have our signed and encoded transaction, we can broadcast it using
[sendrawtransaction](https://chainquery.com/bitcoin-cli/sendrawtransaction):

`bitcoin-cli sendrawtransaction <transaction>`

If the transaction is broadcasted successfully a transaction id will be returned. In this case it was `fbf084eece2f14a017eabfe7fd7db3ee42f0245560a851f39cf34131caabfcb7`.


## Spend from address (after timeout)

In this example we will spend the funds after X blocks have been mined.

```py
# we are continuing the code from above

txid = lx("1f55451b6fe0d839ada38724d00a94118afedaf6564e492887493e3c9dd0dca4")
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
destination = CBitcoinAddress("bcrt1q3qk7a5d2963feda6d6lrrvrvsnp05zlzasme8f").to_scriptPubKey() 
txout = CMutableTxOut(amount_less_fee, destination)

# The default nSequence of FFFFFFFF won't let you redeem when there's a CHECKTIMELOCKVERIFY
txin.nSequence = 0

# Create the unsigned transaction.
tx = CMutableTransaction([txin], [txout])

# nLockTime needs to be at least as large as parameter of CHECKLOCKTIMEVERIFY for script to verify
tx.nLockTime = redeem_blocknum

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
sig = seckey_sender.sign(sighash) + bytes([SIGHASH_ALL])

# Construct a witness for this P2WSH transaction and add to tx.
witness = CScriptWitness([sig, seckey_sender.pub, b'', witness_script])
tx.wit = CTxWitness([CTxInWitness(witness)])

# Done! Print the transaction
print(b2x(tx.serialize()))
# outputs: 010000000001015f8d477e52c7c4338952a9a8a053d682ff42023097e54df102d47f477f3e274900000000000000000001c09ee60500000000160014882deed1aa2ea29cb7ba6ebe31b06c84c2fa0be204483045022100befaddac4d23e7a5edcf5847e6f5cc127021ebc34719b32680bc040aeece5b7402204736e36df03142e7c0573e6d2e9f44e7936e477fd6c76d59a032de62458e6248012103d28046cd12e83832cca3fc4428d254e60092e06fa3f3b8de32062c1b07f58976005b63a8206bd66227651d0fe5c43863d7b29a4097e31dbf51e6603eee75947bff5e96ae438876a914882deed1aa2ea29cb7ba6ebe31b06c84c2fa0be267021701b17576a9141514817c58cc3937e09561f94b6b8e1cb60e92ec6888ac17010000
```

Now that we have our signed and encoded transaction, we can broadcast it using
[sendrawtransaction](https://chainquery.com/bitcoin-cli/sendrawtransaction):

`bitcoin-cli sendrawtransaction <transaction>`

If the transaction is broadcasted successfully a transaction id will be returned. In this case it was `52a580aad3f436096933dd6d3c58dc5131050bdc2d53eea416e934b0d699d89c`.

----

Sources:
- https://bitcoinops.org/en/topics/htlc/
- https://wanwenli.com/blockchain/2018/06/28/Bitcoin-lightning-network.html
- https://github.com/bitcoin/bips/blob/master/bip-0199.mediawiki
- https://min.sc/#c=%2F%2F%20Traditional%20preimage-based%20HTLC%0A%28pk%28A%29%20%26%26%20sha256%28H%29%29%20%7C%7C%20%28pk%28B%29%20%26%26%20older%2810%29%29
- https://en.bitcoin.it/wiki/Hash_Time_Locked_Contracts
- https://medium.com/@jkendzicky16/the-bitcoin-lightning-network-a-technical-primer-d8e073f2a82f#dd0c