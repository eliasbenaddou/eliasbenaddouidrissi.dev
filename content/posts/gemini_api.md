---
title: "Interacting with Cryptocurrency Exchange Gemini's APIs"
date: 2022-06-22
draft: false
slug: ""
tags: ['crypto', 'python', "REST APIs"]
categories: [""]
---

{{< figure src="/images/gemini_logo.png" caption="" width=600 >}}

  <ol>
    <li>
      <a href="#about-the-project">About The Project</a>
    </li>
    <li>
      <a href="#getting-started">Getting Started</a>
      <ul>
        <li><a href="#prerequisites">Prerequisites</a></li>
        <li><a href="#installation">Installation</a></li>
      </ul>
    </li>
    <li><a href="#example-usage">Example Usage</a></li>
        <ul>
        <li><a href="#public-api">Public APIs</a></li>
        <li><a href="#private-api">Private APIs</a></li>
            <ul>
                <li><a href="#creating-orders">Creating Orders</a></li>
                <li><a href="#cancelling-orders">Cancelling Orders</a></li>
                <li><a href="#market-orders-based-on-price-information">Market Orders</a></li>
            </ul>
        </ul>
    <li><a href="#roadmap">Roadmap</a></li>
  </ol>

## About The Project

I recently started working on creating a complete Python project using the some of the best practices when it comes to
writing code, documentation and software engineering. This involved putting together a mostly class based Python wrapper for Gemini's APIs and we'll
be exploring a few examples of it here.

**Github repository**: https://github.com/eliasbenaddou/gemini_api  
**Documentation**: https://eliasbenaddou.github.io/gemini_api/

## Getting Started
### Prerequisites

To get started you'll need a Gemini account and to generate a set of account keys. In this article, we'll be generating Gemini Sandbox API keys which is recommended for familiarising yourself with their APIs before using it on your real money account. You'll need to sign up for a separate Sandbox exchange account on https://exchange.sandbox.gemini.com/.
Navigate to the API section under your account settings and create a pair of public and private keys.
Select the tick boxes to give permissions to your account for Fund Management, Trading and Admin and ensure you keep the keys in a safe place before closing the page.

When provisioning a session key, you have the option of marking the session as "Requires Heartbeat". When selected, if the exchange does not receive a message for 30 seconds, then it will assume there has been an interruption in service and all outstanding orders on this session will be cancelled. To maintain the session, you must send a heartbeat message at a more frequent interval. 

### Installation

Now that we have the access sorted, let's install the Gemini API wrapper:

```python
pip install gemini_api
```

### Example Usage

#### Public API
First let's look at how we can use the package to get some public data on Bitcoin.

```python
from gemini_api.public import Public

public = Public()
public.get_pair_details('btcgbp')
```

```yaml
{'symbol': 'BTCGBP',
 'base_currency': 'BTC',
 'quote_currency': 'GBP',
 'tick_size': 1e-08,
 'quote_increment': 0.01,
 'min_order_size': '0.00001',
 'status': 'open',
 'wrap_enabled': False}
```
The result dictionary contains information about the ticker size and quote increment, which will be useful when creating orders.
The ticker endpoint gives information about recent trading activity and is fetched using the 'get_ticker' function with the trading symbol as the input parameter.

```python
p.get_ticker('btcgbp')
```
```yaml
{'bid': '16288.40',
 'ask': '16301.70',
 'volume': {'BTC': '231.84969379',
  'GBP': '3862517.7357342047',
  'timestamp': 1655929800000},
 'last': '16311.60'}
```

The timestamp is given in Unix format as an integer, and can be converted into a regular date format.

To get candle data for a regular time interval of 30 minutes, we can use the 'get_candles' method
and parse in the time frame as '30m'. We'll also manipulate the result into a dataframe, convert the 
date into a Pandas datetime object and order the dataframe by the date.

```python
btc_data = p.get_candles('btcgbp', '30m')
btc_data_df = pd.DataFrame(pp, columns =['date','open','high','low','close','volume'])
btc_data_df['date'] = pd.to_datetime(btc_data_df['date'], unit='ms')
btc_data_df.set_index('date', inplace=True)
btc_data_df.sort_values('date', inplace=True)
print(btc_data_df)
```
<center>{{< figure src="/img/btc_data_df.png" caption="" >}}</center>

#### Private API
##### Creating Orders
Let's look at some examples of creating orders in Gemini's Sandbox exchange environment. This is done by
setting the sandbox parameter in the class instantiation to true when Sandbox API keys are used.

```python
from gemini_api.endpoints.order import Order
from gemini_api.authentication import Authentication

auth = Authentication(
    public_key="XXXXXXXXXX", private_key="XXXXXXXXXX", sandbox=True,
)

x = Order.new_order(
    auth=auth,
    symbol="btcgbp",
    amount="5",
    price="20000",
    side="buy",
    options=["maker-or-cancel"],
)

print(x.order_id)
```

```python
"128459183"
```

An 'Order' object is created and the class method to create a new order is executed with the resulting 'order_id' attribute printed.
The 'options' parameter 'make-or-cancel' means the order will wait to be filled until the price is hit. Gemini offers several different ways to make an order, which we'll cover more of later on.

##### Cancelling Orders

To cancel a order, you need to specify the 'order_id' and the resulting object will bring back the order status with the attribute 'is_cancelled" as True.

```python
x = Order.cancel_order(
    auth=auth,
    order_id="123456789",
)

print(x.is_cancelled)
```

```python
True
```

You can also cancel all orders created in the session at once or all active orders (rincludes orders created outside of the current session) on your account with either of the following

```python
Order.cancel_session_orders(auth=auth)
```

```python
Order.cancel_active_orders(auth=auth)
```

##### Market Orders Based on Price Information

Gemini doesn't allow market orders via the API, however they offer multiple execution options. By default, the option is set to "maker-or-cancel", which only adds liquidity to the order book. If any of the order is filled immediately, the whole order will instead be cancelled before any execution occurs. 

The "immediate-or-cancel" option will remove liquidity and fill whatever it can immediately and cancel any remaining. There is also "fill-or-kill" to fill the entire order immediately or cancel and "auction-only" to add the order to the auction-only book.

We can combine the public API methods to get BTC prices and the private API methods to create buy orders. From https://docs.gemini.com/rest-api/#symbols-and-minimums we'll obtain the tick size and quote currency price increment which we'll use in the script to mimick the behavior of a market order.  

Here, we'll try to make a new order to purchase £200 worth of BTC at just below the current price of BTC by using a factor, which will complete if the price drops down slightly and is already below our defined upper boundary of £20,000.

Using this method is the cheapest way to 'DCA' on the platform as the fee schedule for the API is 0.2% for the 'Maker Fee'. For fee inclusion in our order, we'll multiply the buy size by 0.998 and use a factor of 0.999 to get an execution price just under the current ask.

```python
from datetime import datetime

from gemini_api.endpoints.public import Public
from gemini_api.endpoints.order import Order
from gemini_api.authentication import Authentication

def unix_ts_to_date(timestamp: str) -> str:
    date = datetime.utcfromtimestamp(int(timestamp)).strftime("%Y-%m-%dT%H:%M:%S")
    return str(date)

auth = Authentication(
    public_key="XXXXXXXXXX", private_key="XXXXXXXXXX", sandbox=True,
)

tick_size = 8
quote_currency_price_inc = 2

try:
    btc_ticker = Public().get_ticker(pair="BTCGBP")
    btc_ask = float(btc_ticker.get("ask"))
except Exception as e:
    print(f"Error getting data from Gemini: {e}")

if btc_ask <= 20000:

    factor = 0.999
    buy_size = 200

    execution_price = str(round(btc_ask * factor, quote_currency_price_inc))

    amount = round((buy_size * 0.998) / float(execution_price), tick_size)

    try:
        order = Order.new_order(
            auth=auth,
            symbol="btcgbp",
            amount=str(amount),
            price=str(btc_ask),
            side="buy",
            options=["maker-or-cancel"],
        )

    except (KeyError, AttributeError):
        print("Error placing order")

    check_order_id = order.order_id
    order_status = Order.order_status(
        auth=auth, order_id=check_order_id, include_trades=False
    )

    if order_status.is_cancelled is False:
        date = unix_ts_to_date(order.timestamp)
        print(f"New order created on {date}UTC with order_id: {order.order_id}")
    else:
        print(f"The order {order.order_id} has been cancelled")
     
```

## Roadmap

The future releases will work on adding Master API key usage, Websockets for market data, OAuth2.0 authentication using tokens and approved address APIs. 