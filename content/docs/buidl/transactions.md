---
title: "Transactions"
weight: 2
---

{{< tip >}}
Library supports mainnet, testnet and signet and requires internet to work. 
{{< /tip >}}

Despite this section being called Transactions, it will actually show how to generate specific
types of Bitcoin addresses and then make a transaction to spend the funds that each specific
address type will hold.

Bitcoin doesn't actually have transaction types but has different types of addresses. Each type of
address has a different way of "unlocking" its funds. The way that dictates how to unlock funds is
set (or scripted) at the time when the address is generated.

Bellow are a different types of addresses and way how to spend funds from those addresses. In the
code samples we will be using **RegTest** network which makes trying out things faster as we can
generate blocks on demand.


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

## P2WSH

P2WSH is an abbreviation for Pay to Witness Script Hash. P2WSH is the **native Segwit** version of
a P2SH. 

{{< tip >}}
"Script Hash addresses" are intended for multisig or other "smart contract" address. If all
you wish to do is receive payment to an address (without multisig) it's better to use P2WPKH as
it's cheaper to spend from those addresses.
{{< /tip >}}

P2WSH has the same semantics as P2SH, except that the signature is not placed at the same location
as before. Segregated Witness (SegWit) moves the proof of ownership from the scriptSig part of the
transaction to a new part called the witness of the input. Script Hash allows you to lock coins to
the hash of a script, and you then provide that original script when you come unlock those coins.

scriptPubKey: `0` `<witnessScriptHash>`

witnessScriptHash: `sha256(pubKey OP_CHECKSIG)`

{{< tip "warning" >}}
Below are two types of examples; for **singlesig** and **multisig**. Singlesig means only one 
signature can unlock the funds in the address and multisig means multiple signatures are needed to
unlock the funds.

The example for singlesig should only serve as an example. We don't recommend using it in the real
world because it is not its intention to be used for addresses with single signatures.
{{< /tip >}}


### Generate address (singlesig)

```py
import hashlib

from buidl.ecc import PrivateKey, Signature
from buidl.helper import decode_base58, big_endian_to_int
from buidl.bech32 import decode_bech32, encode_bech32_checksum
from buidl.script import P2PKHScriptPubKey, RedeemScript, WitnessScript, P2WPKHScriptPubKey
from buidl.tx import Tx, TxIn, TxOut
from buidl.witness import Witness

h = hashlib.sha256(b'correct horse battery staple').digest()
private_key = PrivateKey(secret=big_endian_to_int(h), network="signet")

# Create a witnessScript and corresponding redeemScript. Similar to a scriptPubKey
# the redeemScript must be satisfied for the funds to be spent.
redeem_script = WitnessScript([private_key.point.sec(), 0xac])

address = redeem_script.address("signet")
print('Address:', str(address))
# outputs: tb1qgatzazqjupdalx4v28pxjlys2s3yja9gr3xuca3ugcqpery6c3sqtuzpzy
```

### Spend from address (singlesig)

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

txid = bytes.fromhex("f24d3d8c85ded6d0fbe898a09a2c9f8a8388e4edcf139e52c8714814d85f8273")
vout = 0

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

sig1 = tx.get_sig_segwit(0, private_key, witness_script=redeem_script)

tx.check_sig_segwit(
    0,
    private_key.point,
    Signature.parse(sig1[:-1]),
    witness_script=redeem_script,
)

txin.witness = Witness([sig1, redeem_script.raw_serialize()])

print(tx.serialize().hex())
# outputs: 0100000000010173825fd8144871c8529e13cfede488838a9f2c9aa098e8fbd0d6de858c3d4df20000000000ffffffff01b882010000000000160014003f808a60b27baa380da9f62122e170b294058b0247304402201b812e3a58b18bf83ee65db660af469708583073beaecbd4d7147757068e5ece022034a0b51dc40cdbcba1362f544a467e3dc605395f5e0029d5d2e342aea1ddfac20123210378d430274f8c5ec1321338151e9f27f4c676a008bdf8638d07c0b6be9ab35c71ac00000000
```

Now that we have our signed and encoded transaction, we can broadcast it using
[sendrawtransaction](https://chainquery.com/bitcoin-cli/sendrawtransaction):

`bitcoin-cli sendrawtransaction <transaction>`

If the transaction is broadcasted successfully a transaction id will be returned. In this case it
was `19e8dc2d719e14bc652bda4809007a72c17bdee6e174d4c02f570c48cad691cd`.

### Generate address (multisig)

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

# Create a witnessScript and corresponding redeemScript. Similar to a scriptPubKey
# the redeemScript must be satisfied for the funds to be spent.
redeem_script = WitnessScript(
    [0x52, private_key1.point.sec(), private_key2.point.sec(), 0x52, 0xAE]
)

address = redeem_script.address("signet")
print('Address:', str(address))
# outputs: tb1qljlyqaexx4mmhpl66e6nqdtagjaht87pghuq6p0f98a765c9uj9susmlvt
```

### Spend from address (multisig)

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

txid = bytes.fromhex("e810379ffa5ca20a30f2210d93517aef28d3f20e20b920190161d4f6491a0903")
vout = 0

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

tx = Tx(1, [txin], [txout], 0, network="signet", segwit=True)

sig1 = tx.get_sig_segwit(0, private_key1, witness_script=redeem_script)
sig2 = tx.get_sig_segwit(0, private_key2, witness_script=redeem_script)

tx.check_sig_segwit(
    0,
    private_key.point,
    Signature.parse(sig1[:-1]),
    witness_script=redeem_script,
)

tx.check_sig_segwit(
    0,
    private_key.point,
    Signature.parse(sig2[:-1]),
    witness_script=redeem_script,
)

txin.finalize_p2wsh_multisig([sig1, sig2], redeem_script)

print(tx.serialize().hex())
# outputs: 0100000000010103091a49f6d461011920b9200ef2d328ef7a51930d21f2300aa25cfa9f3710e80000000000ffffffff01b882010000000000160014706385687f40be0af4d91a18b7d62f7c2f934ea90400483045022100b24100d90fdd15d3e694789106f780c4c13f4929ec5cc82445418bedac9dfc93022038f5cc4ade88b41f1398d5f1e31a9d4f9d861f67aa24dff30a6d20e509b591c801483045022100fa9f78c7769010a57aa96939b32fad0d216a1dcf3a1f44a09f0e7b29f56b773402207a795dad938599c920d864001b454ae2f44358f5ac098e913049ea7fd9925cb001475221038d19497c3922b807c91b829d6873ae5bfa2ae500f3237100265a302fdce87b052103d3a9dff5a0bb0267f19a9ee1c374901c39045fbe041c1c168d4da4ce0112595552ae00000000
```

Now that we have our signed and encoded transaction, we can broadcast it using
[sendrawtransaction](https://chainquery.com/bitcoin-cli/sendrawtransaction):

`bitcoin-cli sendrawtransaction <transaction>`

If the transaction is broadcasted successfully a transaction id will be returned. In this case it
was `6064651b405e0e1d6b5cdc0056c3deda11af3593ca64d43b1cd4abff45b7376b`.


## P2PKH

P2PKH is an abbreviation for Pay to Public Key Hash.

scriptPubKey: `OP_DUP` `OP_HASH160` `<pubKeyHash>` `OP_EQUALVERIFY` `OP_CHECKSIG`

### Generate address

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


## P2SH

P2SH is an abbreviation for Pay to Script Hash. It allows you to lock coins to the hash of a
script, and you then provide that original script when you come unlock those coins.

{{< tip >}}
"Script Hash addresses" are intended for multisig or other "smart contract" address. If all
you wish to do is receive payment to an address (without multisig) it's better to use P2WPKH as
it's cheaper to spend from those addresses.
{{< /tip >}}

scriptPubKey: `OP_HASH160` `<scriptHash>` `OP_EQUAL`

{{< tip "warning" >}}
Below are two types of examples; for **singlesig** and **multisig**. Singlesig means only one 
signature can unlock the funds in the address and multisig means multiple signatures are needed to
unlock the funds.

The example for singlesig should only serve as an example. We don't recommend using it in the real
world because it is not its intention to be used for addresses with single signatures.
{{< /tip >}}


### Generate address (singlesig)

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

### Spend from address (singlesig)

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

### Generate address (multisig)

`todo`

### Spend from address (multisig)

`todo`


## P2TR

todo

### Generate address

`todo`

### Spend from address

`todo`

--------------

Sources:
- https://learnmeabitcoin.com
- https://wiki.trezor.io
- https://bitcoindev.network