# Revault transactions

All transactions are version 2 and use version 0 native Segwit scripts.

We use [miniscript](http://bitcoin.sipa.be/miniscript/) and descriptors to generally
describe outputs spending policies.

We denote `N` the number of participants, `M` the number of managers (the subset
of the participants allowed to unlock the unvault transaction output along with the
cosigning servers), and `X` the CSV value in the unvault transaction.

- [vault_tx](#vault_tx)
- [unvault_tx](#unvault_tx)
- [spend_tx](#spend_tx)
- [cancel_tx](#cancel_tx)
- [emergency_txs](#emergency_txs)
    - [vault_emergency_tx](#vault_emergency_tx)
    - [unvault_emergency_tx](#unvault_emergency_tx)


## vault_tx

The deposit transaction.

#### IN

N/A

#### OUT

At least one output paying to `vault_descriptor`, with:
```
vault_descriptor = wsh(vault_witness_script)
vault_witness_script = thresh(N, pubkey1, pubkey2, ..., pubkeyN)
```


## unvault_tx

The transaction which spends the [`vault_tx`](vault_tx) deposit output, and creates an
unvault output only spendable by the managers (along with the cosigning servers)
after `X` blocks.

- version: 2
- locktime: 0

#### IN

- count: 1
- inputs[0]:
    - txid: `<vault_tx txid>`
    - vout: N/A
    - sequence: `0xffffffff`
    - scriptSig: `<empty>`
    - witness: `satisfy(vault_descriptor)`

#### OUT

- count: 2
- outputs[0]:
    - value: `<vault_tx output value - tx_fee - 330>`
    - scriptPubkey: `unvault_descriptor`

- outputs[1]:
    - value: `330`
    - scriptPubkey: `unvault_cpfp_descriptor`

With:
```
unvault_descriptor = wsh(unvault_witness_script)
unvault_witness_script = and(thresh(len(managers), managers), or(1@thresh(len(non_managers), non_managers), 10@and(thresh(len(cosigners), cosigners), older(X))))
```

```
unvault_cpfp_descriptor = wsh(unvault_cpfp_witness_script)
unvault_cpfp_witness_script = thresh(M, pubkey1, pubkey2, ..., pubkeyM)
```

## spend_tx

The transaction which spends the unvaulting transaction by the [`M` + cosigners]
path, only spendable after `X` blocks.

- version: 2
- locktime: 0

#### IN

- count: 1
- inputs[0]:
    - txid: `<unvault_tx txid>`
    - sequence: `X`
    - scriptSig: `<empty>`
    - witness: `satisfy(unvault_descriptor)`

#### OUT

- count: 1
- outputs[0]:
    - value: `<unvault_tx outputs[0] value - tx_fee>`
    - scriptPubkey: N/A


## cancel_tx

This transaction spends the `unvault_tx` using the N-of-N path and pays back to a
vault txout (it is therefore another deposit transaction).

- version: 2
- locktime: 0

#### IN

- count: 1
- inputs[0]:
    - txid: `<unvault_tx txid>`
    - vout: 0
    - sequence: `0xfffffffe`
    - scriptSig: `<empty>`
    - witness: `satisfy(unvault_descriptor)`

#### OUT

- count: 1
- outputs[0]:
    - value: `<unvault_tx outputs[0] value - tx_fee>`
    - scriptPubkey: `vault_descriptor`


## emergency_txs

Emergency transactions are used as deterrents against threat. They lock coins to what we
call an EDV (Emergency Deep Vault): a script chosen by the participants and kept
obfuscated by the properties of P2(W)SH, as the emergency transactions are never meant
to be used.


### vault_emergency_tx

This transaction spends the `vault_tx` to the EDV.

- version: 2
- locktime: 0

#### IN

- count: 1
- inputs[0]:
    - txid: `<vault_tx txid>`
    - vout: N/A
    - sequence: `0xfffffffe`
    - scriptSig: `<empty>`
    - witness: `satisfy(vault_descriptor)`

#### OUT

- count: 1
- outputs[0]:
    - value: `<vault_tx output value - fees>`
    - scriptPubkey: `0x00 SHA256(<EDV_script>)`


### unvault_emergency_tx

This transaction spends the `unvault_tx` to the EDV.

- version: 2
- locktime: 0

#### IN

- count: 1
- inputs[0]:
    - txid: `<unvault_tx txid>`
    - vout: 0
    - sequence: `0xfffffffe`
    - scriptSig: `<empty>`
    - witness: `satisfy(unvault_descriptor)`

#### OUT

- count: 1
- outputs[0]:
    - value: `<unvault_tx outputs[0] value - fees>`
    - scriptPubkey: `0x00 SHA256(<EDV_script>)`
