# perfi
defi portfolio analytics

## About
An open source (AGPLv3), local/private tool to help track your defi degen escapades.

The current perfi functionality is focused on generating a "close enough" estimate of cost basis and loss/gain/income calculations from certain EVM-compatible chain transactions. We are generating a (US IRS) [8949](https://www.irs.gov/forms-pubs/about-form-8949)-style output that can be used as a starting point for tax estimation purposes.

We built this because we couldn't find any tool that could remotely understand DeFi protocols/primitives and how they would map into disposals/income calculations. As far as we know, this is the most advanced DeFi accounting analysis tool in existence and has been tested against real wallets with tens of thousands of transactions interacting with hundreds of contracts.

## ALPHA RELEASE
This software is currently an alpha release. Features may be missing, broken, or confusing. There is no friendly UI yet (outside of a CLI and a generated spreadsheet output), the documentation is sparse, and anyone using this should be familiar with Python.

## Disclaimer
This software should not be used in lieu of accounting / tax review. While we believe that the results generated by this software are useful, they are almost guaranteed to be incorrect without manual adjustments. Also, for your convenience, here's the Disclaimer of Warranty from our LICENSE:

```
THERE IS NO WARRANTY FOR THE PROGRAM, TO THE EXTENT PERMITTED BY
APPLICABLE LAW.  EXCEPT WHEN OTHERWISE STATED IN WRITING THE COPYRIGHT
HOLDERS AND/OR OTHER PARTIES PROVIDE THE PROGRAM "AS IS" WITHOUT WARRANTY
OF ANY KIND, EITHER EXPRESSED OR IMPLIED, INCLUDING, BUT NOT LIMITED TO,
THE IMPLIED WARRANTIES OF MERCHANTABILITY AND FITNESS FOR A PARTICULAR
PURPOSE.  THE ENTIRE RISK AS TO THE QUALITY AND PERFORMANCE OF THE PROGRAM
IS WITH YOU.  SHOULD THE PROGRAM PROVE DEFECTIVE, YOU ASSUME THE COST OF
ALL NECESSARY SERVICING, REPAIR OR CORRECTION.
```

## Requirements
You should be familiar with and have [Python 3.10](https://www.python.org/) and [Poetry](https://python-poetry.org/) installed.

perfi currently depends on several third party API providers (no configuration is required by default):
* [DeBank OpenAPI](https://open.debank.com/) - provides a helpful list of transaction history per chain
* [CoinGecko API](https://www.coingecko.com/en/api) - provides day-resolution coin prices. No API key is required for but requests will be rate-limited. perfi caches and retries so your initial fetches will be slow, but it should eventually work
    * [Paid API plans](https://www.coingecko.com/en/api/pricing) are supported and can be entered in the initial setup


## Getting Started
Here's how to install:
```
git clone https://github.com/AUGMXNT/perfi
cd perfi
poetry install
```

And how to run:
```
# Interactive initial perfi setup (entity, accounts, API keys)
poetry run python bin/cli.py setup

# OPTIONAL: You can also add entities or addresses manually
#
# Add an entity 'peepo'
# > poetry run python bin/cli.py entity create peepo
#
# Add an ethereum-style account named 'degen wallet'
# > poetry run bin/cli.py entity add_address peepo 'degen wallet' 'ethereum' '0x0000000000000000000000000000000000000000'
#
# You add as many wallets as you want to an entity

# Update the Coingecko price token list
poetry run python bin/update_coingecko_pricelist.py

# OPTIONAL: Import data from exchanges (more docs below)
# If you have data from Coinbase, Coinbase Pro, Kraken, Gemini, or Bitcoin.tax you can import this into perfi as well
# > poetry run python bin/import_from_exchange.py --entity_name peepo --file peepo-coinbase-2021-rawtx.csv --exchange coinbase --exchange_account_id peepo

# Import on-chain transactions
poetry run python bin/import_chain_txs.py peepo

# Generate tx/price asset mappings - this step is key for matching like-kind assets and making sensical output
poetry run python bin/map_assets.py

# Turn raw exchange/onchain txs into grouped logical/ledger txs
poetry run python bin/group_transactions.py peepo

# Calculate costbasis lots, disposals, and income
poetry run python bin/calculate_costbasis.py peepo

# Generate 8949 xlsx file
poetry run python bin/generate_8949.py peepo
```

`bin/cli.py` should let you do what you need for updating logical and ledger transactions (updatings prices, transactions types). We try to be smart about updating downstream results, although if things look wonky, you may need to re-run `bin/group_transactions.py` on down...

We've included `--help` for some of the options in the various CLI apps as there's some functionality not in this document yet.

Disposals in generated 8949 xlsx sheets internally link to their associated lots with transaction ids and hashes. LibreOffice and Google Sheets have been tested to play nice with our output file.


We also recommend [DB Browser for SQLite](https://sqlitebrowser.org/) for spelunking around in `data/perfi.db`

## perfi Tax Behavior
This is currently hard-coded. Here's a summary:

* We do specific-id lot matching, generally HIFO (although we do lowest cost basis for loan repayments), we don't do long-term/short-term optimizations, but the HIFO approach should still give near optimal (minimized) tax exposure for US tax rates
* Transfers (including CEX transfers, bridging) are assumed to not disposal or income
* Wrapping/unwrapping is also considered non-disposal
* We will create zero-cost basis lots (with a flag) if necessary
* Single-staking or single-asset deposits are treated as deposits, not disposal
* Swaps/LP are like trades and are treated disposals
* We try to do a good job accounting for income from yield or interest (see Tests)
* We do a best effort for pricing of LP tokens and try to track receipt tokens properly
* For more details atm, take a look at `perfi/costbasis.py:process()` `perfi/models.py:TxLogical.refresh_type()`
  * current tx types: borrow, repay, deposit, withdraw, disposal, lp, swap, yield
  * we have flags for: receipt, ownership_change, disposal, and income


## Notes
* perfi runs as much on your local system as possible, and while we use some third party APIs to make our job easier, our goal is to minimize any PII stored/leaked remotely. We appreciate our privacy and we think others do too.
* This software is licensed with the AGPLv3, a strong copyleft license. This is meant to make sure that end-users of the software will always be able to have control and be able to modify this software to their liking, and as devs, we can maintain optionality/minimize free-riding
* If there's demand/we don't get bored, we may add some (privacy-first) freemium services (higher resolution price/other metadata), access to archival nodes, etc, in the future. In the meantime, if you find this useful or want to support development, donations are gratefully accepted (see below)
* For the exchange imports we've implemented, our HIFO lot matching algorithm almost exactly matches the best-in-class services we've tested (Bitcoin.tax) and beats others like Koinly or CoinTracking which fall down/have bugs (aggressively zero cost-basis transfers, not understanding foreign currency fiat transactions, etc)
* Turns out this stuff is sort of complex, though. We will be publishing some Architectural Decision Records in due time discussing how we handle transactions and cost basis and more
* We think it'll be useful to build a DDL and shared repository to allow community members to extend perfi's ability to understand arbitrary DeFi protocols and strategies
* While the focus of our initial release, taxes are not actually so interesting. Future plans for perfi focus more on the *perf* part:
  * Tracking actual performance of DeFi investment strategies (performance vs benchmarks/hodling, accounting for transaction costs, tax efficiency)
  * Farming/claim calculations and helpers
  * You can see a preview of some of that sort of thing here: https://github.com/AUGMXNT/frax-analysis/

## Caveats
* Somes types of defi transactions still aren't handled well (balancer/Curve-style multi-asset LP entries, maybe some more exotic hedging/margin strategies aren't accounted for, DSA/EOA-style operations, like Instadapp, UniV3 multicalls)
* We don't support NFTs very well atm, [sorry](https://twitter.com/DanielitoG25/status/1498358636648800257)
* Tax treatments are hard coded and may not match your tax regime/preferences. In future versions we plan on making this easier to personalize/configure
* Only supports some EVM chains atm
* Doesn't account for fees atm (this is high on the priority list)

## Did You Know?
This is definitely **NOT** tax advice, but in the US, if you [file an extension](https://www.irs.gov/forms-pubs/extension-of-time-to-file-your-tax-return), and don't make an adequate prepayment, you will owe a [Failure to Pay Penalty](https://www.irs.gov/payments/failure-to-pay-penalty) of 0.5%/mo (3% for the 6 month automatic extension period == 6% APR). [Interesting, right?](https://coindix.com/?kind=stable&sort=-apy&tvl=1m)
* Note, the [Failure to File Penalty](https://www.irs.gov/payments/failure-to-file-penalty) is 5%/mo if you don't get things sorted by the time your extension period ends

## Donations
If you've found this software useful, feel free to zap us some coins/tokens. We promise to farm the shit out of it:
```
0x0bcee2cd8564b2c61caec20113f1f87a16e10cb0
```


## See also
* [DeBank](https://debank.com/) - this is by far the best DeFi portfolio viewer and we use their [OpenAPI](https://open.debank.com/) extensively. You should definitely be using it.
* [rotki](https://github.com/rotki/rotki) - this app shares some goals with perfi, and would have saved a few hundred hours and counting of dev time if it supported the chains/protocols we needed. There's a small dedicated team that [have been working for years](https://github.com/rotki/rotki) - worth a look if it'll do what you need. They have a GUI, [real documentation](https://rotki.readthedocs.io/en/latest/index.html), and like [users and stuff](https://github.com/rotki/rotki/issues)
* [bitcoin.tax](https://bitcoin.tax/) - if all you have is centralized exchange transactions, crypto taxes are a solved problem. We checked out over a dozen services and bitcoin.tax did the best job of any of them (but most of them are probably good enough).
