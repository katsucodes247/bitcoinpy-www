---
title: "P2SH address (multisig)"
weight: 5
---

{{< tip "warning" >}}
The example for 1-of-1 should only serve as an example. We don't recommend using it in the real
world because it is not its intention. Instead of 1-of-1 use P2PKH!
{{< /tip >}}

## Generate address (1-of-1)

```py
import hashlib

from buidl.ecc import PrivateKey, Signature
from buidl.bech32 import decode_bech32
from buidl.helper import decode_base58, big_endian_to_int, SIGHASH_ALL, int_to_byte
from buidl.script import P2PKHScriptPubKey, RedeemScript, Script, P2WPKHScriptPubKey
from buidl.tx import Tx, TxIn, TxOut

h = hashlib.sha256(b'correct horse battery staple').digest()
private_key = PrivateKey(secret=big_endian_to_int(h), network="signet")

# Create a redeemScript. Similar to a scriptPubKey the redeemScript must be satisfied for the funds
# to be spent.
redeem_script = RedeemScript([private_key.point.sec(), 0xac])
address = redeem_script.address("signet")
print("Address", address)
# outputs: Address: 2Msc7itHhx2x8MEkTthvtED9pFC36J7QpQb
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

txid = bytes.fromhex("65665709c7d83b7efb8e58f5c6e08a3ebefd752dd83d7af9e0022b13388c3780")
vout = 1

# Specify the amount send to your P2WSH address.
COIN = 100000000
amount = int(0.001 * COIN)

# Calculate an amount for the upcoming new UTXO. Set a high fee (5%) to bypass bitcoind minfee
# setting on regtest.
amount_less_fee = int(amount * 0.99)

# Create the txin structure, which includes the outpoint. The scriptSig defaults to being empty as
# is necessary for spending a P2WSH output.
txin = TxIn(txid, vout)

# Specify a destination address and create the txout.
h160 = decode_bech32("tb1qqqlcpznqkfa65wqd48mzzghpwzefgpvtvl0a7k")[2]
txout = TxOut(amount=amount_less_fee, script_pubkey=P2WPKHScriptPubKey(h160))

tx = Tx(1, [txin], [txout], 0, network="signet") #, segwit=True)

sighash = tx.sig_hash_legacy(0, redeem_script)

# signed sighash
signed_sighash = private_key.sign(sighash).der() + int_to_byte(SIGHASH_ALL)

txin.script_sig = Script([signed_sighash, redeem_script.raw_serialize()])

tx.check_sig_legacy(
    0,
    private_key.point,
    Signature.parse(signed_sighash[:-1]),
    redeem_script=redeem_script,
)

# Done! Print the transaction
print(tx.serialize().hex())
# outputs: 010000000180378c38132b02e0f97a3dd82d75fdbe3e8ae0c6f5588efb7e3bd8c709576665010000006c473044022040048e65bb5a58c69d9f514472b5d581f83105422210c134edf20c2ba1d2be50022029069037cc852b83b341a397c0a753bbaf6aefee9e9455483ffaa39666784ad80123210378d430274f8c5ec1321338151e9f27f4c676a008bdf8638d07c0b6be9ab35c71acffffffff01b882010000000000160014003f808a60b27baa380da9f62122e170b294058b00000000
```

Now that we have our signed and encoded transaction, we can broadcast it using
[sendrawtransaction](https://chainquery.com/bitcoin-cli/sendrawtransaction):

`bitcoin-cli sendrawtransaction <transaction>`

If the transaction is broadcasted successfully a transaction id will be returned. In this case it
was `9987191906843b5c99218cbae7f73d0ae85c0b62b78fdaf9755eb6a4a9856858`.

## Generate address (2-of-2)

In this example we show how to create a 2-of-2 multisig address. This means that two signatures are
required in order to unlock funds.

```py
import hashlib

from buidl.ecc import PrivateKey, Signature
from buidl.helper import decode_base58, big_endian_to_int
from buidl.bech32 import decode_bech32, encode_bech32_checksum
from buidl.script import P2PKHScriptPubKey, RedeemScript, WitnessScript, P2WPKHScriptPubKey
from buidl.tx import Tx, TxIn, TxOut
from buidl.witness import Witness

# first signature
h = hashlib.sha256(b'correct horse battery staple first').digest()
private_key1 = PrivateKey(secret=big_endian_to_int(h), network="signet")

# second signature
h = hashlib.sha256(b'correct horse battery staple second').digest()
private_key2 = PrivateKey(secret=big_endian_to_int(h), network="signet")

# Create a redeem script
redeem_script = RedeemScript.create_p2sh_multisig(
    quorum_m=2,
    pubkey_hexes=[
        private_key1.point.sec().hex(),
        private_key2.point.sec().hex(),
    ],
    sort_keys=False,
)

address = redeem_script.address("signet")
print('Address:', str(address))
# outputs: 2N3avDJKpr9c8pkRSYgWAsHSVCmRX3ce3w7
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

txid = bytes.fromhex("20a58d3d90b0a9758f2e717dfb4195f270323221b8be01b06c95551807654b9c")
vout = 1

# Specify the amount send to your P2WSH address.
COIN = 100000000
amount = int(0.001 * COIN)

# Calculate an amount for the upcoming new UTXO. Set a high fee (5%) to bypass bitcoind minfee
# setting on regtest.
amount_less_fee = int(amount * 0.99)

# Create the txin structure, which includes the outpoint. The scriptSig defaults to being empty as
# is necessary for spending a P2WSH output.
txin = TxIn(txid, vout)

# Specify a destination address and create the txout.
h160 = decode_bech32("tb1qwp3c26rlgzlq4axergvt04300shexn4f56q5f7")[2]
txout = TxOut(amount=amount_less_fee, script_pubkey=P2WPKHScriptPubKey(h160))

tx = Tx(1, [txin], [txout], 0, network="signet") # , segwit=True)

sig1 = tx.get_sig_legacy(0, private_key1, redeem_script=redeem_script)
sig2 = tx.get_sig_legacy(0, private_key2, redeem_script=redeem_script)

tx.check_sig_segwit(
    0,
    private_key1.point,
    Signature.parse(sig1[:-1]),
    redeem_script=redeem_script,
)

tx.check_sig_segwit(
    0,
    private_key2.point,
    Signature.parse(sig2[:-1]),
    redeem_script=redeem_script,
)

txin.finalize_p2sh_multisig([sig1, sig2], redeem_script)

print(tx.serialize().hex())
# outputs: 01000000019c4b65071855956cb001beb821323270f29541fb7d712e8f75a9b0903d8da52001000000da004730440220470aef78655414f73ca5ae0feec073d49a134ba1bc3b3ea85d2aa864a9735a66022029382bd5f497dc8435f75dff4ebe6cb7205064738487bbfd7099fd3741c61bc901483045022100aa867aeba637ef0aa8f6386e06bcce5010afb081087ab4e3d01ce610f57eaa4702204b50096f8225cac48a93cddab25be1463b160c77605b93556cc215ed83c0d0f301475221038d19497c3922b807c91b829d6873ae5bfa2ae500f3237100265a302fdce87b052103d3a9dff5a0bb0267f19a9ee1c374901c39045fbe041c1c168d4da4ce0112595552aeffffffff01b882010000000000160014706385687f40be0af4d91a18b7d62f7c2f934ea900000000
```

Now that we have our signed and encoded transaction, we can broadcast it using
[sendrawtransaction](https://chainquery.com/bitcoin-cli/sendrawtransaction):

`bitcoin-cli sendrawtransaction <transaction>`

If the transaction is broadcasted successfully a transaction id will be returned. In this case it
was `8439bee332441a97a5bd01396c95353a26a163287e12bd691e1940b734f77478`.

