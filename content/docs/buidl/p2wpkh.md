---
title: "P2WPKH address"
weight: 2
---

P2WPKH is an abbreviation for Pay to Witness Public Key Hash. P2WPKH is the **native Segwit** 
version of a P2PKH.

P2WPKH has the same semantics as P2PKH, except that the signature is not placed at the same
location as before. Segregated Witness (SegWit) moves the proof of ownership from the scriptSig
part of the transaction to a new part called the witness of the input. 

scriptPubKey: `0` `<witnessPubKeyHash>`

## Generate address

```py
import hashlib

from buidl.ecc import PrivateKey, Signature
from buidl.helper import decode_base58, big_endian_to_int
from buidl.bech32 import decode_bech32, encode_bech32_checksum
from buidl.script import P2PKHScriptPubKey, RedeemScript, WitnessScript, P2WPKHScriptPubKey
from buidl.tx import Tx, TxIn, TxOut

h = hashlib.sha256(b'correct horse battery staple').digest()
private_key = PrivateKey(secret=big_endian_to_int(h), network="signet")
address = private_key.point.bech32_address("signet")
print('Address:', str(address))
# outputs: Address: tb1q08alc0e5ua69scxhvyma568nvguqccrvl7rkgx
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

txid = bytes.fromhex("5375b209e02aa4fd665b10fbbfbd5bbe3045a307ddcabd7571f87ab571a41e98")
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

tx = Tx(1, [txin], [txout], 0, network="signet", segwit=True)

redeem_script = private_key.point.p2sh_p2wpkh_redeem_script()

# Sign the transaction (buidl makes a request to the explorer to fetch public key here)
tx.sign_input(0, private_key, redeem_script=redeem_script)

# Done! Print the transaction
print(tx.serialize().hex())
# outputs: 01000000000101981ea471b57af87175bdcadd07a34530be5bbdbffb105b66fda42ae009b275530100000000ffffffff01b882010000000000160014003f808a60b27baa380da9f62122e170b294058b02483045022100ca58968430508ce0a36b45c7ad53de85fbf111569ed822d69844fc232df9a7a6022011445083ce81f4cce1e8bc4a82b8338f9025de080dfb7e202352fe66411d117f01210378d430274f8c5ec1321338151e9f27f4c676a008bdf8638d07c0b6be9ab35c7100000000
```

Now that we have our signed and encoded transaction, we can broadcast it using
[sendrawtransaction](https://chainquery.com/bitcoin-cli/sendrawtransaction):

`bitcoin-cli sendrawtransaction <transaction>`

If the transaction is broadcasted successfully a transaction id will be returned. In this case it was `d1be307fe1d50d1e0044ba6b031cf0ad7d105d4d7d1d1cc6a4abff8dd225386f`.
