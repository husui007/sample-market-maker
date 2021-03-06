# BitMEX Market Maker

This is a sample market making bot for use with [BitMEX](https://www.bitmex.com).

It is free to use and modify for your own strategies. It provides the following:

* A `BitMEX` object wrapping the REST and WebSocket APIs.
  * All data is realtime and efficiently [fetched via the WebSocket](market_maker/ws/ws_thread.py). This is the fastest way to get market data.
  * Orders may be created, queried, and cancelled via `BitMEX.buy()`, `BitMEX.sell()`, `BitMEX.open_orders()` and the like.
  * Withdrawals may be requested (but they still must be confirmed via email and 2FA).
  * Connection errors and WebSocket reconnection is handled for you.
  * [Permanent API Key](https://testnet.bitmex.com/app/apiKeys) support is included.
* [A scaffolding for building your own trading strategies.](#advanced-usage)
  * Out of the box, a simple market making strategy is implemented that blankets the bid and ask.
  * More complicated strategies are up to the user. Try incorporating [index data](https://testnet.bitmex.com/app/index/.XBT),
    query other markets to catch moves early, or develop your own completely custom strategy.

**Develop on [Testnet](https://testnet.bitmex.com) first!** Testnet trading is completely free and is identical to the live market.

> BitMEX is not responsible for any losses incurred when using this code. This code is intended for sample purposes ONLY - do not
  use this code for real trades unless you fully understand what it does and what its caveats are.

> This is not a sophisticated market making program. It is intended to show the basics of market making while abstracting some
  of the rote work of interacting with the BitMEX API. It does not make smart decisions and will likely lose money.

## Getting Started

1. Create a [Testnet BitMEX Account](https://testnet.bitmex.com) and [deposit some TBTC](https://testnet.bitmex.com/app/deposit).

### Non-developer instructions

The below instructions are fine for a non-developer, but you will not be installing this fork of the Bitmex market-maker. They are fine for
getting your feet wet:

1. Install: `pip install bitmex-market-maker`. It is strongly recommeded to use a virtualenv.
1. Create a marketmaker project: run `marketmaker setup`
    * This will create `settings.py` and `market_maker/` in the working directory.
    * Modify `settings.py` to tune parameters.

### Developer instructions

If you wish to follow my changes and/or make changes, you will need to do the following.

1. `git clone` my repo
1. `cp ./market_maker/_settings_base.py settings.py`

### Continuing, both non-developers and developers should:

1. Edit settings.py to add your [BitMEX API Key and Secret](https://testnet.bitmex.com/app/apiKeys) and change bot parameters.
    * Note that user/password authentication is not supported.
    * Run with `DRY_RUN=True` to test cost and spread.
1. Run it: `marketmaker [symbol]`
1. Satisfied with your bot's performance? Create a [live API Key](https://www.bitmex.com/app/apiKeys) for your
   BitMEX account, set the `BASE_URL` and start trading!

## Operation Overview

This market maker works on the following principles:

* The market maker tracks the last `bidPrice` and `askPrice` of the quoted instrument to determine where to start quoting.
* Based on parameters set by the user, the bot creates a descriptions of orders it would like to place.
  - If `settings.MAINTAIN_SPREADS` is set, the bot will start inside the current spread and work outwards.
  - Otherwise, spread is determined by interval calculations.
* If the user specifies position limits,(via `CHECK_POSITION_LIMITS, MIN_POSITION` and
`MAX_POSITION` these are checked. If the current position is beyond a limit,
  the bot stops quoting that side of the market. **IMPORTANT**: these parameters can save you
  from buying/selling more contracts that your margin can sustain. If you dont use them, then
  you may well margin yourself into a drained account.
* These order descriptors are compared with what the bot has currently placed in the market.
  - If an existing order can be amended to the desired value, it is amended.
  - Otherwise, a new order is created.
  - Extra orders are canceled.
* The bot then prints details of contracts traded, tickers, and total delta.

## Simplified Output

The following is some of what you can expect when running this bot:

```
2016-01-28 17:29:31,054 - INFO - market_maker - BitMEX Market Maker Version: 1.0
2016-01-28 17:29:31,074 - INFO - ws_thread - Connecting to wss://testnet.bitmex.com/realtime?subscribe=quote:XBT7D,trade:XBT7D,instrument,order:XBT7D,execution:XBT7D,margin,position
2016-01-28 17:29:31,074 - INFO - ws_thread - Authenticating with API Key.
2016-01-28 17:29:31,075 - INFO - ws_thread - Started thread
2016-01-28 17:29:32,079 - INFO - ws_thread - Connected to WS. Waiting for data images, this may take a moment...
2016-01-28 17:29:32,079 - INFO - ws_thread - Got all market data. Starting.
2016-01-28 17:29:32,079 - INFO - market_maker - Using symbol XBT7D.
2016-01-28 17:29:32,079 - INFO - market_maker - Order Manager initializing, connecting to BitMEX. Live run: executing real trades.
2016-01-28 17:29:32,079 - INFO - market_maker - Resetting current position. Cancelling all existing orders.
2016-01-28 17:29:33,460 - INFO - market_maker - XBT7D Ticker: Buy: 388.61, Sell: 389.89
2016-01-28 17:29:33,461 - INFO - market_maker - Start Positions: Buy: 388.62, Sell: 389.88, Mid: 389.25
2016-01-28 17:29:33,461 - INFO - market_maker - Current XBT Balance: 3.443498
2016-01-28 17:29:33,461 - INFO - market_maker - Current Contract Position: -1
2016-01-28 17:29:33,461 - INFO - market_maker - Avg Cost Price: 389.75
2016-01-28 17:29:33,461 - INFO - market_maker - Avg Entry Price: 389.75
2016-01-28 17:29:33,462 - INFO - market_maker - Contracts Traded This Run: 0
2016-01-28 17:29:33,462 - INFO - market_maker - Total Contract Delta: -17.7510 XBT
2016-01-28 17:29:33,462 - INFO - market_maker - Creating 4 orders:
2016-01-28 17:29:33,462 - INFO - market_maker - Sell 100 @ 389.88
2016-01-28 17:29:33,462 - INFO - market_maker - Sell 200 @ 390.27
2016-01-28 17:29:33,463 - INFO - market_maker -  Buy 100 @ 388.62
2016-01-28 17:29:33,463 - INFO - market_maker -  Buy 200 @ 388.23
-----
2016-01-28 17:29:37,366 - INFO - ws_thread - Execution: Sell 1 Contracts of XBT7D at 389.88
2016-01-28 17:29:38,943 - INFO - market_maker - XBT7D Ticker: Buy: 388.62, Sell: 389.88
2016-01-28 17:29:38,943 - INFO - market_maker - Start Positions: Buy: 388.62, Sell: 389.88, Mid: 389.25
2016-01-28 17:29:38,944 - INFO - market_maker - Current XBT Balance: 3.443496
2016-01-28 17:29:38,944 - INFO - market_maker - Current Contract Position: -2
2016-01-28 17:29:38,944 - INFO - market_maker - Avg Cost Price: 389.75
2016-01-28 17:29:38,944 - INFO - market_maker - Avg Entry Price: 389.75
2016-01-28 17:29:38,944 - INFO - market_maker - Contracts Traded This Run: -1
2016-01-28 17:29:38,944 - INFO - market_maker - Total Contract Delta: -17.7510 XBT
2016-01-28 17:29:38,945 - INFO - market_maker - Amending Sell: 99 @ 389.88 to 100 @ 389.88 (+0.00)

```

## Advanced usage

You can implement custom trading strategies using the market maker. `market_maker.OrderManager`
controls placing, updating, and monitoring orders on BitMEX. To implement your own custom
strategy, subclass `market_maker.OrderManager` and override `OrderManager.place_orders()`:

```
from market_maker.market_maker import OrderManager

class CustomOrderManager(OrderManager):
    def place_orders(self) -> None:
        # implement your custom strategy here
```

Your strategy should provide a set of orders. An order is a dict containing price, quantity, and
whether the order is buy or sell. For example:

```
buy_order = {
    'price': 1234.5, # float
    'orderQty': 100, # int
    'side': 'Buy'
}

sell_order = {
    'price': 9876.5, # float
    'orderQty': 100, # int
    'side': 'Sell'
}
```

Call `self.converge_orders()` to submit your orders. `converge_orders()` will create, amend,
and delete orders on BitMEX as necessary to match what you pass in:

```
def place_orders(self) -> None:
    buy_orders = []
    sell_orders = []

    # populate buy and sell orders, e.g.
    buy_orders.append({'price': 998.0, 'orderQty': 100, 'side': "Buy"})
    buy_orders.append({'price': 999.0, 'orderQty': 100, 'side': "Buy"})
    sell_orders.append({'price': 1000.0, 'orderQty': 100, 'side': "Sell"})
    sell_orders.append({'price': 1001.0, 'orderQty': 100, 'side': "Sell"})

    self.converge_orders(buy_orders, sell_orders)
```

To run your strategy, call `run_loop()`:
```
order_manager = CustomOrderManager()
order_manager.run_loop()
```

Your custom strategy will run until you terminate the program with CTRL-C. There is an example
in `custom_strategy.py`.

## Notes on Rate Limiting

By default, the BitMEX API rate limit is 300 requests per 5 minute interval (avg 1/second).

This bot uses the WebSocket and bulk order placement/amend to greatly reduce the number of calls sent to the BitMEX API.

Most calls to the API consume one request, except:

* Bulk order placement/amend: Consumes 0.1 requests, rounded up, per order. For example, placing 16 orders consumes
  2 requests.
* Bulk order cancel: Consumes 1 request no matter the size. Is not blocked by an exceeded ratelimit; cancels will
  always succeed. This bot will always cancel all orders on an error or interrupt.

If you are quoting multiple contracts and your ratelimit is becoming an obstacle, please
[email support](mailto:support@bitmex.com) with details of your quoting. In the vast majority of cases,
we are able to raise a user's ratelimit without issue.

## Troubleshooting

Common errors we've seen:

* `TypeError: __init__() got an unexpected keyword argument 'json'`
  * This is caused by an outdated version of `requests`. Run `pip install -U requests` to update.

## Compatibility

This module supports Python 3.5 and later.

## See also

BitMEX has a Python [REST client](https://github.com/BitMEX/api-connectors/tree/master/official-http/python-swaggerpy)
and [websocket client.](https://github.com/BitMEX/api-connectors/tree/master/official-ws/python)

### Related software

* [Krypto](https://github.com/ctubio/Krypto-trading-bot)
* [Tribeca](https://github.com/michaelgrosner/tribeca)
* [Autoview](https://www.youtube.com/watch?v=1xXOc0y6wRg)


# THEORY and PRACTICE of Market-Making

I recommend [this article](https://rados.io/how-profitable-is-market-making-on-different-exchanges/) if you are not familiar with market-making and its pitfalls. Reading that will make my discussion eaiser to follow.

## Daily range

The most important idea for you to grasp *now* about market-making is that you are setting
up a grid of buy and sell orders and you make money on the daily fluctuation of the market.

As you can see from [this picture](http://take.ms/Q9kOY) having a grid of buy and sell orders
allows you to profit from the daily fluctuations of bitcoin regardless of whether the price
is going up or down.

The caveat is: you need to need to be 100% certain that your buy/sell grid covers the range
of fluctuation **AND** that you have enough capital in your account to make the buys/sells
throughout an entire day of fluctuation. Check our TOTAL capital versus AVAILABLE credit
after setting up your buy/sell grid and make sure that only 50% of your capital is expended
in opening up these limit orders.

An important concept is daily price range. You must know the daily price range of Bitcoin
and be certain that your buy/sell grid covers it. For instance, [here you can see that I have
a wide range of orders on the book](http://take.ms/nEm0N) and the daily range of Bitcoin is 4 times or more smaller than my grid.

Reading an article like [this Bloomberg one](https://www.bloomberg.com/news/articles/2018-05-02/bitcoin-s-daily-trading-range-falls-from-4-700-to-124-chart) will help you understand
what range your grid should be prepared to cover on a daily basis.

For me, all I do is look at the UPside and DOWNside contracts at Bitmex to get an idea of
the range that Bitcoin will trade in - this is because these future contracts are designed
to not pay back and the prices of these contracts are calculated to make sure that Bitcoin
does not rise or fall to those values within a week.

For instance, at this moment, Bitcoin is trading at $7639.00 and the UP contract strike is
$8250 and the DOWN contract strike is $6750. So as long as my grid covers that price range,
I am good to go for a week.

## Are you bullish on Bitcoin?

If you are bullish on Bitcoin and don't want to get caught with sell orders when BTC
skyrockets to the moon without notice, then you may want to only offer buy-side market-making
instead of two-way market making.

The way to do this is to enable `CHECK_POSITION_LIMITS` and then set `MIN_POSITION` to `-1`
so that you never hold more than 1 short contract. Set the `MAX_POSITION` to a number that
you feel comfortable with. e.g., if your initial buy contract is for 500 contracts and your
orders are spaced 1% apart and you want to be able to buy all the way down to BTC dropping
20% in value, then you need 20 * 500 as your `MAX_POSITION` and no more.

## CHECK_POSITION_LIMITS should be True by default

You do not want to wake (as I have) and find that a 0.6BTC account has been drained because there was no limit on total amount of contracts you could have open.

## Monitoring your position

Keep an eye on your TOTAL balance, AVAILABLE balance and the Liquidation price for your contracts, as [this picture shows](http://take.ms/sdNZO) then use
a chart to understand what the maximum trading range for your instrument (XBTUSD in most cases) is and make sure the difference between the current price and the liquidation price is
at least twice that. [This diagram](http://take.ms/p6ZX7) shows a simple study of daily trading range to give me a sense of assurance.
