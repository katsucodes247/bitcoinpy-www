---
title: "Regtest Guide"
weight: 2
draft: true
---

We assume you're running a Bitcoin node on Regtest. In case you're not and you're not sure how to configure your Bitcoin node to run on Regtest [have a look at this guide](/docs/running-bitcoin/#regtest).

#### Generate address

```bash
bitcoin-cli getnewaddress
```

#### Generate (mine) blocks

`bitcoin-cli generatetoaddress <number of blocks> <youraddress>`

```bash
bitcoin-cli generatetoaddress 50 bcrt1qywrq947s5yjjn7ye45y9wwh3g093cvec5fl9gn
```

Note that after genering blocks the "mining" rewards can not be usued until they have matured. Rewards are considered matured after 100 confirmations. Our above example command generated 50 blocks which means the rewards are still immature so you'll need to generate more blocks.