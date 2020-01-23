# How to use Dune Analytics  

### With Dune Analytics you can: 

1. #### Query human-readable smart contract data with PostgreSQL.
2. #### Visualize your findings.
3. #### Create dashboards and share them with public links.
4. #### Explore analysis created by other community members. You can fork any query by the click of a button.  


Here are some tips and tricks on how to get started with the data and interface. Create a user for free at [duneanalytics.com](https://www.duneanalytics.com/).

If you have any questions feel free to join our [Telegram channel](https://t.me/dunefriends) or email us hello@duneanalytics.com.

## Table of contents

[TOC]

## Data tables

You can currently query from these databases:

1. **Ethereum mainnet**
    * [Raw Ethereum data](#raw-ethereum-data)
    * [Decoded smart contract data](#decoded-data)
    * [Centralised exchanges trading data](#Centralised-exchanges-trading-data) 
2. **Binance DEX**

### Ethereum Mainnet

The most commonly used tables are

* **Event data** for each project where you typically get the action that occured in the smart contract. Simply search for project name to find the relevant tables.
* **Ethereum transactions** which you use to get a time dimension`block_time`. Typically join with event on `transactions.hash = evt_tx_hash`.
* **Prices.usd** which gives you USD price of various assets per minute. Typically joined on minute with the event data and multiplied by the on-chain value (trade, transfer etc).
* **Erc20.tokens** gives you contract address, symbol and number of decimals for popular tokens. Token value transfers are then divided by `10` to the power of `decimals` from this table.


#### Raw Ethereum data
 
* Blocks
* Logs
* Transactions
* Receipts
* Traces

You probably won't use this too much when doing analysis with Dune (see [decoded data](#decoded-data)), but it's always nice to have just in case. 

You can [click here](https://ethereum.stackexchange.com/questions/268/ethereum-block-architecture)  to learn more about Ethereum's data structure, but again it's not really needed for using Dune.

### Decoded smart contract data

Instead of working with the traces, logs and receipts we transform smart contract activity to nice human readable type tables.

You will find a table for each `event` and `call` to and from a smart contract.

These are named as `projectName.contractName_evt_eventName` for events and `projectName.contractName_call_eventName` for calls. 

For example `uniswap.Exchange_evt_AddLiquidity` or `uniswap.Exchange_call_addLiquidity`

Using the event tables is usually sufficient, but in some cases you will want to use the `call` tables. For instance Maker DAO which don't give you too many events you can use tables like `maker.SaiTub_call_draw`.


**We have decoded data for the most popular smart contract projects. Let us know [here](https://docs.google.com/forms/d/e/1FAIpQLSe1DjWbrjpkV7RleAqKC3td7t1ajugTyPlBXbS4q6GKaMWYtQ/viewform?usp=sf_link) if you have a request for decoding of data.** 

#### Several instances of the same contract

We decode the smart contract data based on the bytecode, so all contracts with the same bytecode end up in the same table. The column `contract_address` shows you exactly which smart contract the data is for. Because of this you can do one `SELECT` statement across all Uniswap contracts for instance. If you want to look at only a single contract you filter by that address. 

#### 

### Centralised exchanges trading data 

Token volume is great, but more often than not you want to know the USD value of smart contract activity.

You can easily get and join that information with onchain data using the data we pull from the coincap API. See how to use it below.

* prices.usd
    * contract_address
    * price
    * minute
    * symbol (ticker)

Price is the volume-weighted price based on real-time market data every minute, translated to USD.

The data is fetched from this [Coincap API](https://docs.coincap.io/?0=o&1=n&2=b&3=o&4=a&5=r&6=d&7=i&8=n&9=g&10=c&11=o&12=m&13=p&14=l&15=e&16=t&17=e&version=latest#61e708a8-8876-4fb2-a418-86f12f308978).

## How to work with the data

You can interact with the [data tables](#Data-tables) through our interface at [duneanalytics.com](https://www.duneanalytics.com/).

To create a new query you simply click Create -> Query


On your left you can select which database you want to use in the dropdown list and then see the data tables in the window. Just search for the project you are interested in working with.

### Tips you might find handy

#### Remove decimals

Ether transfers and most ERC-20 tokens have 18 decimal places. To get a more human readable number you need to remove all the decimals. The table `erc20.tokens` gives you contract address, symbol and number of decimals for popular tokens. Token value transfers are then divided by 10 to the power of decimals from this table: `transfer_value / 10^erc20.tokens.decimals`

#### Using Inline Ethereum addresses

In Dune Ethereum addresses are stored as postgres bytearrays which are encoded with the `\x` prefix. This differs from the customary `0x` prefix. If you'd like to use an inline address, say to filter for a given token, you would do
```sql
WHERE token = '\x89d24a6b4ccb1b6faa2625fe562bdd9a23260359'
```
which is simply short for 
``` sql
WHERE token = '\x89d24a6b4ccb1b6faa2625fe562bdd9a23260359'::bytea
```

#### Join with transactions table to get time

If you want to get a time dimension for your query you can join an event or call table with `ethereum.transactions` on `evt_tx_hash`.

We've added `block_time` to the `ethereum.transactions` table for your convenience. A neat way to use it is with the `date_trunc` function like this 

```SQL
SELECT date_trunc('week', tx.block_time) AS time
```
Here you can use minute, day, week, month.

#### How to get USD price

To get the USD volume of onchain activity you typically want to join the smart contract event you are looking at with the usd price and join on minute.
 
``` SQL
LEFT JOIN prices.usd p ON p.minute = 
date_trunc('minute', tx.block_time)
```

and make sure that asset matches asset

 `AND b."asset" = p.contract_address`

Then you can simply multiply the value or amount from the smart contract event with the usd price in your `SELECT` statement: `* p.price`. 



#### Token symbols

You often want to group your results by token address. For instance you want to see volume on a DEX grouped by token. However, a big list of token addresses are abstract and hard to digest. Therefore you often want to use the token symbol instead. Simply join the table `coincap.symbols` with your event table where asset = token address. You then select symbol in your select statement instead of token address.

**NB** The coincap table cointains tokens that are traded on centralised exchanges. If you are working with more obscure tokens you should be careful with joining with this table because tokens that are not in the coincap table might be excluded from your results. 

## Visualize and share your findings

When you think your query is correct and are ready to give your results a beautiful visual graph you simply click *+ New Visualization* next to your results table.

Most often you want the Visualization Type to be either a Chart (for values over time and/or grouped by something) or a Counter (with a single important metric).


## Working with specific projects

### Uniswap

When working with Uniswap data you typically want to know which token each exchange address belongs to. For the event `uniswap.Exchange_evt_EthPurchase` the field `address` tells you what exchange address the EthPurchase happened on. You can then join with the event `uniswap.Factory_evt_NewExchange` on the `exchange` field to see which token that exchange belongs to. 

See [token symbols](#token-symbols) above on how to turn token addresses into symbols in your results.


### Compound v2

There are no events emitted when new token contracts are deployed with Compound v2. The Compound v2 markets therefore need to be manually mapped to different token symbols.

The easiest way to work with the Compound v2 data is to insert the relevant token information into your query with this snippet:

```sql
WITH temp_token_info (address, symbol, decimals, underlying_symbol, underlying_decimals) AS
    (VALUES
    ('\x6c8c6b02e7b2be14d4fa6022dfd6d75921d90e4e', 'cBAT', 8, 'BAT', 18),
    ('\xf5dce57282a584d2746faf1593d3121fcac444dc', 'cDAI', 8, 'DAI', 18),
    ('\x4ddc2d193948926d02f9b1fe9e1daa0718270ed5', 'cETH', 8, 'ETH', 18),
    ('\x158079ee67fce2f58472a96584a73c7ab9ac95c1', 'cREP', 8, 'REP', 18),
    ('\x39aa39c021dfbae8fac545936693ac917d5e7563', 'cUSDC', 8, 'USDC', 6),
    ('\xb3319f5d18bc0d84dd1b4825dcde5d5f7266d407', 'cZRX', 8, 'ZRX', 18),
    ('\xC11b1268C1A384e55C48c2391d8d480264A3A7F4', 'cWBTC', 8, 'WBTC', 8)
    )
```

Then you can join address from `temp_token_info` with address from the event `compoundv2."cEther_evt_Borrow"` for instance. Thanks to Dune user pyggie for inspiring this snippet.

## Explore community analysis

You can explore what analysis other community members have created with Dune. Simply go to queries or dashboards in the top left of the Dune Analytics interface. 

If you are interested in a specific project (for instance 0x or Augur) or a domain (for instance DeFi or gaming) simply search for it in the top tight corner and you'll only see results related to your search term.

If there are no results you might need to get your hands dirty and create the analysis yourself :)  

# Change log

#### [9th of January 2020](https://hackmd.io/YOP3YIgaRAejTPE190sOjw?view) - ERC20 transfer table, Fallback decoding and more

# Dashboard ideas (Work in progress)

## General ideas

| Idea  | Status  |Link| Idea by  | Brought to life by  |
|---|---|---|---|---|
| Input your address and get ETH spent on gas  | :white_check_mark:  | [Here](https://explore.duneanalytics.com/queries/603?p_eth_address=11b6a5fe2906f3354145613db0d99ceb51f604c9)  | @sniko_  |@sniko_|
|   |   |   |   ||


## By project

See dashboard ideas for various projects and tips on how to 

* [0x](https://hackmd.io/xTfDymtlTAiYdQF-YIj1ug)
* [IDEX](https://hackmd.io/byDBLLhoRhmaNYb7NCZwNw)
* [Numerai](https://hackmd.io/2c97h6XuQCqfDHrfWDI4Bg#)

# Some sample queries

## Growth rate

Assuming you have a query with a count of some event grouped by time (month for instance) you can add this snippet to you `select` statement to get growth rate.

```SQL
(count(distinct event) - lag(count(distinct event), 1)
over (order by date_trunc('month', tx.block_time))) / lag(count(distinct action), 1)
over (order by date_trunc('month', tx.block_time)) as "Growth rate"
```

Multiply the number by 100 to get percentage.

## Users and amount over a trailing period

```sql
WITH tx AS (SELECT hash, block_time FROM ethereum.transactions
WHERE block_time > now() - interval '7 days'
) --We limit the transactions first to avoid joining with the full transactions table

SELECT  date_trunc('day', tx.block_time),
        COUNT (DISTINCT buyer),
        SUM(eth_bought / 1e18)
FROM uniswap."Exchange_evt_EthPurchase" p
LEFT JOIN tx
ON p.evt_tx_hash = tx.hash
GROUP BY 1
ORDER BY 1;
```
## USD value of token utilised for an event

```sql
SELECT
date_trunc('week', tx.block_time),
SUM(amount/1e18 * p.price) AS staked
FROM numerai."SimpleGriefing_evt_StakeAdded" s --Replace with relevant event
LEFT JOIN ethereum.transactions tx ON tx.hash = s.evt_tx_hash
LEFT JOIN prices.usd p ON p.minute = date_trunc('minute', tx.block_time)
WHERE p.symbol = 'NMR' --Replace with relevant token
GROUP BY 1;
```



## US trading volume per token over time

```sql
SELECT    price.symbol,
          date_trunc('week', tx.block_time) AS time,
          SUM(["FilledAmount"] /10^(erc.decimals) * price.price) AS usd_value
   FROM [table].["fill_event"] filled
   LEFT JOIN ethereum.transactions tx ON filled.tx_hash = tx.hash
   LEFT JOIN prices.usd price ON date_trunc('minute', tx.block_time) = price.minute AND [tokenAddress] = price.contract_address
   LEFT JOIN erc20.tokens erc ON erc.contract_address = "takerAsset"
    WHERE tx.block_time < date_trunc('week', current_date)::date
   GROUP BY 1,
            2;
```

## Retention

```sql
WITH new_user_activity AS
  (SELECT activity.*
   FROM
     ( /*Here we get user address from the relevant event table
        and join with tx.block_time*/
     SELECT src AS address,
             date_trunc('week', tx.block_time) AS date
      FROM maker."DAI_evt_Transfer" t
      INNER JOIN ethereum.transactions tx ON t.tx_hash = tx.hash
      
      WHERE tx.block_time > date_trunc('week', current_date)::date - 35 
    AND tx.block_time < date_trunc('week', current_date)::date
      ) AS activity
   JOIN
     (SELECT address,
             MIN(date) AS first_time
      FROM
        ( /*Getting the same event as above*/
        SELECT src AS address,
             date_trunc('day', tx.block_time) AS date
      FROM maker."DAI_evt_Transfer" t
      INNER JOIN ethereum.transactions tx ON t.tx_hash = tx.hash) AS senders
      GROUP BY address
      ) AS users  
    ON users.address = activity.address
   AND users.first_time = activity.date),
   
 cohort_active_user_count AS (
  SELECT 
    date, count(distinct address) AS count 
  FROM new_user_activity
  GROUP BY 1
)

SELECT date, period, new_users, retained_users, retention 
FROM (
  SELECT 
    new_user_activity.date AS date,
     extract('epoch' from (future_activity.date - new_user_activity.date)) / 604800 AS period,
    max(cohort_size.count) as new_users, -- all equal in group
    count(distinct future_activity.address) AS retained_users,
    count(distinct future_activity.address) / 
      max(cohort_size.count)::float as retention
  FROM new_user_activity
  LEFT JOIN
       (SELECT src AS address,
             date_trunc('week', tx.block_time) AS date
      FROM maker."DAI_evt_Transfer" t
      INNER JOIN ethereum.transactions tx ON t.tx_hash = tx.hash
      WHERE tx.block_time > date_trunc('week', current_date)::date - 35 
    AND tx.block_time < date_trunc('week', current_date)::date)  as future_activity on
    new_user_activity.address = future_activity.address
    AND new_user_activity.date < future_activity.date
    AND (new_user_activity.date + interval '5 weeks')
      >= future_activity.date
  LEFT JOIN cohort_active_user_count as cohort_size on 
    new_user_activity.date = cohort_size.date 
  GROUP BY 1, 2) t
WHERE period is NOT  NULL
ORDER BY date, period
```

# Misc.

## New data structure Oct 2019

If you are forking old queries you need to be aware of these changes to you can replace old table and column names with new ones. 

### USD price tables
Previously the usd price table was named `coincap."tokens/usd"` this is now changed to `prices.usd` with the fields

```SQL
symbol -- unchanged
contract_address -- previously "asset"
price -- previously "average"
minute -- previously "time"
```

### Dune generated columns

We add some data fields to all decoded tables for `calls` and `events`. These are now named `evt_index, evt_tx_hash, contract_address`. Previously these were simplt named `index, tx_hash, address` but we wanted to make it clear that these are added by Dune and not extracted directly from the blockchain. 


# Known issues

### Function overloading

We have a known issue with *function overloading*. There are a few cases where smart contract developers use function overloading, i.e. specify two functions with the same name but different parameters. In these cases, we will currently only have _one_ of the implementations in our database. We’re working on a fix for this. One known case is the two approve implementations in the SAI contract.



