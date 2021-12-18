---
title: "P2PKH address"
weight: 4
---

## Generate address

```py
import hashlib

from buidl.ecc import PrivateKey, Signature
from buidl.bech32 import decode_bech32
from buidl.helper import decode_base58, big_endian_to_int
from buidl.script import P2PKHScriptPubKey, RedeemScript, WitnessScript, P2WPKHScriptPubKey
from buidl.tx import Tx, TxIn, TxOut

h = hashlib.sha256(b'correct horse battery staple').digest()
private_key = PrivateKey(secret=big_endian_to_int(h), network="signet")
address = private_key.point.address(network="signet")
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

txid = bytes.fromhex("06633b187444530eb74284730e45c00946b7f83c4acca5213037e44406b0dceb")
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

tx = Tx(1, [txin], [txout], 0, network="signet")

# Sign the transaction (buidl makes a request to the explorer to fetch public key here)
tx.sign_p2pkh(0, private_key)

# Done! Print the transaction
print(tx.serialize().hex())
# outputs: 0100000001ebdcb00644e4373021a5cc4a3cf8b74609c0450e738442b70e534474183b6306010000006a4730440220026f57ed7143868dba36b4bc3123719a9a4bb65a479ab2e086de9f93e4c43f1c02204943c6e5ead4de886aa1b4777cffa072a21c8d4c13743543d1a4e1f06ab28d0301210378d430274f8c5ec1321338151e9f27f4c676a008bdf8638d07c0b6be9ab35c71ffffffff01b882010000000000160014003f808a60b27baa380da9f62122e170b294058b00000000
```

Now that we have our signed and encoded transaction, we can broadcast it using
[sendrawtransaction](https://chainquery.com/bitcoin-cli/sendrawtransaction):

`bitcoin-cli sendrawtransaction <transaction>`

If the transaction is broadcasted successfully a transaction id will be returned. In this case it
was `ee002c8a2d9a791382600c5e9526faf173c60827abde4f34dd2267d480aa66bd`.
