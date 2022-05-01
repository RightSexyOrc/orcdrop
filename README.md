# ORC Vested Token Spend Bundle

You can use the spend bundle to check the status and ownership of your
tokens, and to make your tokens appear in your wallet after the 3 months
(414720 blocks) vesting period has passed. You'll need the following command
line tools:

- `cdv` and `brun` [https://pypi.org/project/chia-dev-tools/]
- `jq` [https://stedolan.github.io/jq/]

## Checking the status of the coin

```sh
$ cdv rpc coinrecords --by puzhash "$(jq '.["coin_spends"][0]["coin"]["puzzle_hash"]' spendbundle_$address_$amount.json | tr -d '"')"
```

Look for the `amount` key in the json output. Its value divided by 1000 is the
amount of tokens in the coin. `confirmed_block_index` gives the block index, on
which this coin was created.

## Checking ownership of the coin

For this we need to extract the puzzle reveal from the spend bundle. We'll store
in a variable in our shell session.

```sh
puzzle_reveal="$(jq '.["coin_spends"][0]["puzzle_reveal"]' spendbundle_$address_$amount.json | tr -d '"')"
```

Then we proceed in three steps:

1. Is this a coin for a CAT (Chia Asset Token)?

```sh
$ cdv clsp treehash "$(cdv clsp uncurry "$puzzle_reveal" | sed -n 2p)"
```

If the output is `72dec062874cd4d3aab892a0906688a1ae412b0109982e1797a170add88bdcdc` then the answer is *yes*.

2. Is this an ORC CAT coin?

```sh
$ cdv clsp uncurry "$puzzle_reveal" | sed -n 5p
```

If the output is `- 0xc1a3f20688d704dd96f1c00405fb731ef0564a3e601a5a9449d6bb285597f163` then the answer is *yes*.

3. Will the tokens from this coin show up in your wallet after the vesting period?

We run the inner puzzle with the solution "()":

```sh
brun "$(cdv clsp uncurry "$puzzle_reveal" | sed -n 6p | sed 's/^..//')" "()"
```

The output should be something like this:

```
((82 0x065400) (51 0x<your encoded address> 0x<amount> (0x<your encoded address>)))
```

- `(82 0x065400)` means that spending this coin will succeed only after 414720
(0x065400 in hexadecimal) blocks have passed since this coin
`confirmed_block_index`.
- `(51 0x<your encoded address> <amount> (0x<your encoded address>))` means that
the given amount will be sent to the address.

Running `cdv encode <your encoded address>` should output *your address*.

(Optional) You can check that the solution for the inner puzzle in the spend
bundle is actually the solution we used for running the inner puzzle.

```sh
brun "2" "$(cdv clsp disassemble "$(jq '.["coin_spends"][0]["solution"]' spendbundle_$address_$amount.json | tr -d '"')")"
```

## Make the tokens appear in your wallet

Simply run:

```sh
$ cdv rpc pushtx spendbundle_$address_$amount.json
```

Before the vesting period has passed the output will be:
```
{
    "status": "PENDING",
    "success": true
}
```

Unless the tokens already show up in your wallet, after the vesting has passed
the output will be:
```
{
    "status": "SUCCESS",
    "success": true
}
```

## References
[Coins, Spends and Wallets](https://chialisp.com/docs/coins_spends_and_wallets/)
