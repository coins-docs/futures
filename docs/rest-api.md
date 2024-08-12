---
title: "Rest-Api"
permalink: /rest-api/
layout: default
nav: sidebar/rest-api.html

---


# Public Rest API for Coins Futures (2024-08-31)

## General API Information

* The base endpoint is: **https://fapi.coins.xyz**
* All endpoints return data in either a JSON object or array format.
* Data is returned in **ascending** order, with the oldest records appearing first and the newest records appearing last.
* All time and timestamp related fields are expressed in milliseconds.

## HTTP Return Codes
* HTTP `4XX` return codes are used for malformed requests; the issue is on the sender's side.
* HTTP `403` return code is used when the WAF Limit (Web Application Firewall) has been violated.
* HTTP `429` return code is used when breaking a request rate limit.
* HTTP `418` return code is used when an IP has been auto-banned for continuing to send requests after receiving `429` codes.
* HTTP `5XX` return codes are used for internal errors; the issue is on exchange's side. It is important to **NOT** treat this as a failure operation; the execution status is UNKNOWN and could have been a success.

## Error Codes
* Any endpoint can return an ERROR; the error payload is as follows:

```javascript
{
  "code": -1000,
  "msg": "An unknown error occurred while processing the request."
}
```

* Specific error codes and messages are defined in another document.



## General Information on Endpoints

* For `GET` endpoints, parameters must be sent as a `query string`.
* For `POST`, `PUT`, and `DELETE` endpoints, the parameters may be sent as a `query string` or in the `request body` with content type
  `application/x-www-form-urlencoded`. It is also possible to use a combination of parameters in both the query string and the request body if desired.
* Parameters can be sent in any order.
* If a parameter is included in both the `query string` and the `request body`, the value of the parameter from the `query string` will take precedence and be used.



### LIMITS

**General Info on Limits**

* `intervalNum` describes the amount of the interval. For example, `intervalNum 5` with `intervalLetter M` means "Every 5 minutes".

* A HTTP status code 429 will be returned when the rate limit is violated.



### IP Limits

* Each route has a `weight` which determines the number of requests each endpoint counts for. Endpoints with heavier operations or those that involve multiple symbols will have a higher `weight`.
* When a 429 response code is received, it is mandatory for the API user to back off and refrain from making further requests.
* **Repeated failure to comply with rate limits and a disregard for backing off after receiving 429 responses can result in an automated IP ban. The HTTP status code 418 is used for IP bans.**
* IP bans are tracked and their duration increases for repeat offenders, ranging **from 2 minutes to 3 days**.
* A `Retry-After` header iis included in 418 or 429 responses, indicating the number of seconds that need to be waited in order to prevent a ban (for 429) or until the ban is lifted (for 418).
* **The limits imposed by the API are based on IP addresses rather than API keys**



### Order Rate Limits

* When the order count exceeds the limit, you will receive a 429 error without the `Retry-After` header.

* The order rate limit is counted against each IP and UID.



### Websocket Limits

* A single connection can listen to a maximum of 1024 streams.



### /api/ Limit Introduction

* For endpoints related to `/api/*`:

  * There are two modes of limit enforcement: IP limit and UID limit. Each mode operates independently.

  * The IP limit allows a maximum of 1200 requests per minute across all endpoints within the `/api/*` namespace.



### Endpoint Security Type

* Each endpoint is associated with a security type, which indicates how you should interact with it. The security type is specified next to the name of the endpoint.
  * If no security type is mentioned, assume that the security type is NONE.
* API keys are passed to the Rest API via the `X-COINS-APIKEY`header.
* Both API keys and secret keys **are case sensitive**.
* API keys can be configured to have access only to specific types of secure endpoints. For example, one API key may be restricted to TRADE routes only, while another API key can have access to all routes except TRADE.
* By default, API keys have access to all secure routes.

Security Type | Description
------------ | ------------
NONE | Endpoint can be accessed freely.
TRADE | Endpoint requires sending a valid API Key and signature.
USER_DATA | Endpoint requires sending a valid API Key and signature.
USER_STREAM | Endpoint requires sending a valid API Key.
MARKET_DATA | Endpoint requires sending a valid API Key.

* `TRADE` and `USER_DATA` endpoints are `SIGNED` endpoints.



### SIGNED (TRADE and USER_DATA) Endpoint Security

* `SIGNED` endpoints require an additional parameter, `signature`, to be
  sent in the  `query string` or `request body`.
* Endpoints use `HMAC SHA256` signatures. The `HMAC SHA256 signature` is a keyed `HMAC SHA256` operation.
  Use your `secretKey` as the key and `totalParams` as the value for the HMAC operation.
* The `signature` is **not case sensitive**.
* `totalParams` is defined as the `query string` concatenated with the
  `request body`.



### Timing Security

* A `SIGNED` endpoint requires an additional parameter, `timestamp`, to be included in the request. The `timestamp` should be the millisecond timestamp indicating when the request was created and sent.
* An optional parameter, `recvWindow`, can be included to specify the validity duration of the request in milliseconds after the timestamp. If `recvWindow` is not provided, **it will default to 5000 milliseconds**.
* The logic is as follows:

  ```javascript
  if (timestamp < (serverTime + 1000) && (serverTime - timestamp) <= recvWindow) {
    // process request
  } else {
    // reject request
  }
  ```

**Serious trading is about timing.** Networks can be unstable and unreliable,
which can lead to requests taking varying amounts of time to reach the
servers. With `recvWindow`, you can specify that the request must be
processed within a certain number of milliseconds or be rejected by the
server.

**To ensure optimal performance, it is recommended to use a `recvWindow` value of 5000 milliseconds or less. The maximum value allowed for `recvWindow`is 60,000 milliseconds and should not exceed this limit.**



### SIGNED Endpoint Examples for POST /openapi/v1/order

Here is a step-by-step example of how to send a valid signed payload from the
Linux command line using `echo`, `openssl`, and `curl`:

Key | Value
------------ | ------------
apiKey | tAQfOrPIZAhym0qHISRt8EFvxPemdBm5j5WMlkm3Ke9aFp0EGWC2CGM8GHV4kCYW
secretKey | lH3ELTNiFxCQTmi9pPcWWikhsjO04Yoqw3euoHUuOLC3GYBW64ZqzQsiOEHXQS76

Parameter | Value
------------ | ------------
symbol | ETHBTC
side | BUY
type | LIMIT
timeInForce | GTC
quantity | 1
price | 0.1
recvWindow | 5000
timestamp | 1538323200000



#### Example 1: As a query string

* **queryString:** symbol=ETHBTC&side=BUY&type=LIMIT&timeInForce=GTC&quantity=1&price=0.1&recvWindow=5000&timestamp=1538323200000
* **HMAC SHA256 signature:**

```shell
[linux]$ echo -n "symbol=ETHBTC&side=BUY&type=LIMIT&timeInForce=GTC&quantity=1&price=0.1&recvWindow=5000&timestamp=1538323200000" | openssl dgst -sha256 -hmac "lH3ELTNiFxCQTmi9pPcWWikhsjO04Yoqw3euoHUuOLC3GYBW64ZqzQsiOEHXQS76"
(stdin)= 5f2750ad7589d1d40757a55342e621a44037dad23b5128cc70e18ec1d1c3f4c6
```

* **curl command:**

```shell
(HMAC SHA256)
[linux]$ curl -H "X-COINS-APIKEY: tAQfOrPIZAhym0qHISRt8EFvxPemdBm5j5WMlkm3Ke9aFp0EGWC2CGM8GHV4kCYW" -X POST 'https://$HOST/openapi/v1/order?symbol=ETHBTC&side=BUY&type=LIMIT&timeInForce=GTC&quantity=1&price=0.1&recvWindow=5000&timestamp=1538323200000&signature=5f2750ad7589d1d40757a55342e621a44037dad23b5128cc70e18ec1d1c3f4c6'
```



#### Example 2: As a request body

* **requestBody:** symbol=ETHBTC&side=BUY&type=LIMIT&timeInForce=GTC&quantity=1&price=0.1&recvWindow=5000&timestamp=1538323200000
* **HMAC SHA256 signature:**

```shell
[linux]$ echo -n "symbol=ETHBTC&side=BUY&type=LIMIT&timeInForce=GTC&quantity=1&price=0.1&recvWindow=5000&timestamp=1538323200000" | openssl dgst -sha256 -hmac "lH3ELTNiFxCQTmi9pPcWWikhsjO04Yoqw3euoHUuOLC3GYBW64ZqzQsiOEHXQS76"
(stdin)= 5f2750ad7589d1d40757a55342e621a44037dad23b5128cc70e18ec1d1c3f4c6
```

* **curl command:**

```shell
(HMAC SHA256)
[linux]$ curl -H "X-COINS-APIKEY: tAQfOrPIZAhym0qHISRt8EFvxPemdBm5j5WMlkm3Ke9aFp0EGWC2CGM8GHV4kCYW" -X POST 'https://$HOST/openapi/v1/order' -d 'symbol=ETHBTC&side=BUY&type=LIMIT&timeInForce=GTC&quantity=1&price=0.1&recvWindow=5000&timestamp=1538323200000&signature=5f2750ad7589d1d40757a55342e621a44037dad23b5128cc70e18ec1d1c3f4c6'
```



#### Example 3: Mixed query string and request body

* **queryString:** symbol=ETHBTC&side=BUY&type=LIMIT&timeInForce=GTC
* **requestBody:** quantity=1&price=0.1&recvWindow=5000&timestamp=1538323200000
* **HMAC SHA256 signature:**

```shell
[linux]$ echo -n "symbol=ETHBTC&side=BUY&type=LIMIT&timeInForce=GTCquantity=1&price=0.1&recvWindow=5000&timestamp=1538323200000" | openssl dgst -sha256 -hmac "lH3ELTNiFxCQTmi9pPcWWikhsjO04Yoqw3euoHUuOLC3GYBW64ZqzQsiOEHXQS76"
(stdin)= 885c9e3dd89ccd13408b25e6d54c2330703759d7494bea6dd5a3d1fd16ba3afa
```

* **curl command:**

```shell
(HMAC SHA256)
[linux]$ curl -H "X-COINS-APIKEY: tAQfOrPIZAhym0qHISRt8EFvxPemdBm5j5WMlkm3Ke9aFp0EGWC2CGM8GHV4kCYW" -X POST 'https://$HOST/openapi/v1/order?symbol=ETHBTC&side=BUY&type=LIMIT&timeInForce=GTC' -d 'quantity=1&price=0.1&recvWindow=5000&timestamp=1538323200000&signature=885c9e3dd89ccd13408b25e6d54c2330703759d7494bea6dd5a3d1fd16ba3afa'
```

Note that in Example 3, the signature is different from the previous examples. Specifically, there is be no `&` character between `GTC` and `quantity=1`.



## Public API Endpoints

### Terminology

These terms will be used throughout the documentation, so it is recommended especially for new users to read to help their understanding of the API.

* `base asset` refers to the asset that is the `quantity` of a symbol. For the symbol BTCUSDT, BTC would be the `base asset`.
* `quote asset` refers to the asset that is the `price` of a symbol. For the symbol BTCUSDT, USDT would be the `quote asset`.



### ENUM definitions

**Symbol status:**

* TRADING
* BREAK (ongoing)
* CANCEL_ONLY (ongoing)

**Order status:**

Status | Description
-----------| --------------
`NEW` | The order has been accepted by the engine.
`PARTIALLY_FILLED`| A part of the order has been filled.
`FILLED` | The order has been completed.
`PARTIALLY_CANCELED` | A part of the order has been cancelled with self trade.
`CANCELED` | The order has been canceled by user 
`EXPIRED`       | The order has been cancelled by matching-engine: LIMIT FOK order not filled, limit order not fully filled etc 

**Order types:**

* LIMIT
* MARKET
* LIMIT_MAKER
* STOP_LOSS
* STOP_LOSS_LIMIT
* TAKE_PROFIT
* TAKE_PROFIT_LIMIT



**Order Response Type (newOrderRespType):**

* ACK

* RESULT

* FULL



**Order side:**

* BUY
* SELL



**Anti self-trading behaviour(stpFlag):**

| Value | Description                                     |
| ----- | ----------------------------------------------- |
| `CB`  | Both orders will be cancelled by match engine   |
| `CN`  | The new order will be cancelled by match engine |
| `CO`  | The old order will be cancelled by match engine |



**Time in force (timeInForce):**

This sets how long an order will be active before expiration.

Status | Description
-----------| --------------
`GTC` | Good Til Canceled <br> An order will be on the book unless the order is canceled.
`IOC` | Immediate Or Cancel <br> An order will try to fill the order as much as it can before the order expires.
`FOK`| Fill or Kill <br> An order will expire if the full order cannot be filled upon execution.

**Kline/Candlestick chart intervals:**

m -> minutes; h -> hours; d -> days; w -> weeks; M -> months

* 1m
* 3m
* 5m
* 15m
* 30m
* 1h
* 2h
* 4h
* 6h
* 8h
* 12h
* 1d
* 3d
* 1w
* 1M



### Filters

Filters define trading rules on a symbol or an exchange. Filters come in two forms: `symbol filters` and `exchange filters`.



#### Symbol filters

##### PRICE_FILTER

The `PRICE_FILTER` defines the `price` rules for a symbol. There are 3 parts:

* `minPrice` defines the minimum `price`/`stopPrice` allowed.
* `maxPrice` defines the maximum `price`/`stopPrice` allowed.
* `tickSize` defines the intervals that a `price`/`stopPrice` can be increased/decreased by.

In order to pass the `price filter`, the following must be true for `price`/`stopPrice`:

* `price` >= `minPrice`
* `price` <= `maxPrice`
* (`price`-`minPrice`) % `tickSize` == 0

**/exchangeInfo format:**

```javascript
  {
    "filterType": "PRICE_FILTER",
    "minPrice": "0.00000100",
    "maxPrice": "100000.00000000",
    "tickSize": "0.00000100"
  }
```



##### PERCENT_PRICE

The `PERCENT_PRICE` filter defines valid range for a price based on the weighted average of the previous trades. `avgPriceMins` is the number of minutes the weighted average price is calculated over.

In order to pass the `percent price`, the following must be true for `price`:

- `price` <= `weightedAveragePrice` * `multiplierUp`
- `price` >= `weightedAveragePrice` * `multiplierDown`

**/exchangeInfo format:**

```javascript
  {
    "filterType": "PERCENT_PRICE",
    "multiplierUp": "1.3000",
    "multiplierDown": "0.7000",
    "avgPriceMins": 5
  }
```



##### PERCENT_PRICE_SA

The `PERCENT_PRICE_SA` filter defines valid range for a price based on the  simple average of the previous trades. `avgPriceMins` is the number of minutes the simple average price is calculated over.

In order to pass the `percent_price_sa`, the following must be true for `price`:

- `price` <= `simpleAveragePrice` * `multiplierUp`
- `price` >= `simpleAveragePrice` * `multiplierDown`

**/exchangeInfo format:**

```javascript
  {
    "filterType": "PERCENT_PRICE_SA",
    "multiplierUp": "1.3000",
    "multiplierDown": "0.7000",
    "avgPriceMins": 5
  }
```



##### PERCENT_PRICE_BY_SIDE

The `PERCENT_PRICE_BY_SIDE` filter defines the valid range for the price based on the last price of the symbol.
There is a different range depending on whether the order is placed on the `BUY` side or the `SELL` side.

Buy orders will succeed on this filter if:

- `Order price` <= `bidMultiplierUp` * `lastPrice`
- `Order price` >= `bidMultiplierDown` * `lastPrice`

Sell orders will succeed on this filter if:

- `Order Price` <= `askMultiplierUp` * `lastPrice`
- `Order Price` >= `askMultiplierDown` * `lastPrice`

**/exchangeInfo format:**

```javascript
  {
    "filterType": "PERCENT_PRICE_BY_SIDE",
    "bidMultiplierUp": "1.2",
    "bidMultiplierDown": "0.2",
    "askMultiplierUp": "1.5",
    "askMultiplierDown": "0.8",
  }
```



##### PERCENT_PRICE_INDEX

The `PERCENT_PRICE_INDEX` filter defines valid range for a price based on the  index price which is calculated based on  several exhanges in the market by centain rule. (indexPrice wobsocket pushing will be available in future)

In order to pass the `percent_price_index`, the following must be true for `price`:

- `price` <= `indexPrice` * `multiplierUp`
- `price` >= `indexPrice` * `multiplierDown`

**/exchangeInfo format:**

```javascript
  {
    "filterType": "PERCENT_PRICE_INDEX",
    "multiplierUp": "1.3000",
    "multiplierDown": "0.7000",
  }
```



##### PERCENT_PRICE_ORDER_SIZE

The `PERCENT_PRICE_ORDER_SIZE` filter  is used to determine whether the execution of an order would cause the market price to fluctuate beyond the limit price, and if so, the order will be rejected.

In order to pass the `percent_price_order_size`, the following must be true:

- A buy order needs to meet: the market price after the order get filled  <`askPrice` * `multiplierUp`
- A sell order needs to meet: the market price after the order get filled  >`bidPrice` * `multiplierDown`

**/exchangeInfo format:**

```javascript
  {
    "filterType": "PERCENT_PRICE_ORDER_SIZE",
    "multiplierUp": "1.3000",
    "multiplierDown": "0.7000"
  }
```



##### STATIC_PRICE_RANGE

The `STATIC_PRICE_RANGE` filter defines a static valid range for the price.

In order to pass the `static_price_range`, the following must be true for `price`:

- `price` <= `priceUp`
- `price` >= `priceDown`

**/exchangeInfo format:**

```javascript
  {
    "filterType": "STATIC_PRICE_RANGE",
    "priceUp": "520",
    "priceDown": "160"
  }
```



##### LOT_SIZE

The `LOT_SIZE` filter defines the `quantity` (aka "lots" in auction terms) rules for a symbol. There are 3 parts:

* `minQty` defines the minimum `quantity` allowed.
* `maxQty` defines the maximum `quantity` allowed.
* `stepSize` defines the intervals that a `quantity`can be increased/decreased by.

In order to pass the `lot size`, the following must be true for `quantity`:

* `quantity` >= `minQty`
* `quantity` <= `maxQty`
* (`quantity`-`minQty`) % `stepSize` == 0

**/exchangeInfo format:**

```javascript
  {
    "filterType": "LOT_SIZE",
    "minQty": "0.00100000",
    "maxQty": "99999999.00000000",
    "stepSize": "0.00100000"
  }
```



##### NOTIONAL

The `NOTIONAL` filter defines the acceptable notional range allowed for an order on a symbol.

In order to pass this filter, the notional (`price * quantity`) has to pass the following conditions:

- `price * quantity` <= `maxNotional`
- `price * quantity` >= `minNotional`

**/exchangeInfo format:**

```javascript
{
   "filterType": "NOTIONAL",
   "minNotional": "10.00000000",
   "maxNotional": "10000.00000000"
}
```



##### MAX_NUM_ORDERS

The `MAX_NUM_ORDERS` filter defines the maximum number of orders an account is allowed to have open on a symbol.
Note that both triggered "algo" orders and normal orders are counted for this filter.

**/exchangeInfo format:**

```javascript
  {
    "filterType": "MAX_NUM_ORDERS",
    "maxNumOrders": 200
  }
```



##### MAX_NUM_ALGO_ORDERS

The `MAX_ALGO_ORDERS` filter defines the maximum number of untriggered "algo" orders an account is allowed to have open on a symbol.
"Algo" orders are `STOP_LOSS`, `STOP_LOSS_LIMIT`, `TAKE_PROFIT`, and `TAKE_PROFIT_LIMIT` orders.

**/exchangeInfo format:**

```javascript
  {
    "filterType": "MAX_NUM_ALGO_ORDERS",
    "maxNumAlgoOrders": 5
  }
```



#### Exchange Filters

None for now



### General endpoints

#### Test connectivity

```shell
GET /openapi/v1/ping
```

Test connectivity to the Rest API.

**Weight:** 1

**Parameters:** NONE

**Response:**

```javascript
{}
```



#### Check server time

```shell
GET /openapi/v1/time
```

Test connectivity to the Rest API and get the current server time.

**Weight:** 1

**Parameters:** NONE

**Response:**

```javascript
{
  "serverTime": 1538323200000
}
```



#### Get user ip

```shell
GET /openapi/v1/user/ip
```

Get the user ip.

**Weight:** 1

**Parameters:** NONE

**Response:**

```javascript
{
  "ip": "57.181.16.43"
}
```



#### Exchange information

```shell
GET /openapi/v1/exchangeInfo
```

Current exchange trading rules and symbol information

**Weight:** 1

**Parameters:**

| Name    | Type   | Mandatory | Description                                                  |
| ------- | ------ | --------- | ------------------------------------------------------------ |
| symbol  | STRING | NO        | Specify a trading pair, for example symbol=ETHBTC            |
| symbols | STRING | NO        | x-Specify multiple trading pairs, such as symbol=%5B"ETHBTC","BTCUSDT"%5D, note that %5B represents '[' left bracket, %5D represents ']' right bracket. Direct use of the format ["ETHBTC","BTCUSDT"] is not supported as it is not RFC 3986 compliant. |

**Response:**

```json
{
    "timezone": "UTC",
    "serverTime": "1718958414231",
    "exchangeFilters": [],
    "symbols": [
        {
            "symbol": "ETCUSDT",
            "status": "trading",
            "baseAsset": "ETC",
            "quoteAsset": "USDT",
            "orderTypes": [
                "LIMIT",
                "MARKET",
                "STOP",
                "TAKE_PROFIT",
                "STOP_MARKET",
                "TAKE_PROFIT_MARKET"
            ],
            "filters": []
        },
        {
            "symbol": "AXSUSDT",
            "status": "trading",
            "baseAsset": "AXS",
            "quoteAsset": "USDT",
            "orderTypes": [
                "LIMIT",
                "MARKET",
                "STOP",
                "TAKE_PROFIT",
                "STOP_MARKET",
                "TAKE_PROFIT_MARKET"
            ],
            "filters": []
        },
        {
            "symbol": "BCHUSDT",
            "status": "trading",
            "baseAsset": "BCH",
            "quoteAsset": "USDT",
            "orderTypes": [
                "LIMIT",
                "MARKET",
                "STOP",
                "TAKE_PROFIT",
                "STOP_MARKET",
                "TAKE_PROFIT_MARKET"
            ],
            "filters": []
        },
        {
            "symbol": "TESUSDT",
            "status": "trading",
            "baseAsset": "TES",
            "quoteAsset": "USDT",
            "orderTypes": [
                "LIMIT",
                "MARKET",
                "STOP",
                "TAKE_PROFIT",
                "STOP_MARKET",
                "TAKE_PROFIT_MARKET"
            ],
            "filters": []
        },
        {
            "symbol": "BTCUSDT",
            "status": "trading",
            "baseAsset": "BTC",
            "quoteAsset": "USDT",
            "orderTypes": [
                "LIMIT",
                "MARKET",
                "STOP",
                "TAKE_PROFIT",
                "STOP_MARKET",
                "TAKE_PROFIT_MARKET"
            ],
            "filters": []
        }
    ]
}
```



### Futures Trading endpoints

#### Test new order (TRADE)

```shell
POST /openapi/v1/order/test (HMAC SHA256)
```

Test new order creation and signature/recvWindow long.
Creates and validates a new order but does not send it into the matching engine.

**Weight:** 1

**Parameters:**

Same as `POST /openapi/v1/order`

**Response:**

```javascript
{}
```



#### New order (TRADE)

```shell
POST /openapi/v1/order  (HMAC SHA256)
```

Send in a new order.

**Weight:** 1

**Parameters:**

Name | Type | Mandatory | Description
------------ | ------------ | ------------ | ------------
newClientOrderId | STRING | NO | A unique id among open orders. Automatically generated if not sent. Orders with the same `newClientOrderID` can be accepted only when the previous one is filled, otherwise the order will be rejected.
symbol | STRING | YES |
side | ENUM | YES | `BUY` or `SELL`.
isolated | STRING | NO | `true` or `false`, default `false`.
type | ENUM | YES | `LIMIT`, `MARKET`, `STOP`, `TAKE_PROFIT`, `STOP_MARKET`, `TAKE_PROFIT_MARKET`.
timeInForce | ENUM | NO | `GTC`, `FOK`, `IOC`, `POST_ONLY`, default `GTC`.
quantity | DECIMAL | NO | Cannot be sent with `closePosition=true`(Close-All).
price | DECIMAL | NO |
stopPrice | DECIMAL | NO | Used with `STOP/STOP_MARKET` or `TAKE_PROFIT/TAKE_PROFIT_MARKET` orders.
closePosition | STRING | NO | `true` or `false`. Close-All, used with `STOP_MARKET` or `TAKE_PROFIT_MARKET`.
reduceOnly | STRING | NO | `true` or `false`, default `false`. Cannot be sent with `closePosition=true`.
workingType | ENUM | NO | stopPrice triggered by: `MARK_PRICE` or `LAST_PRICE`, default `LAST_PRICE`.
newOrderRespType | ENUM | NO | `ACK` or `RESULT`, default `ACK`.
recvWindow | LONG | NO |
timestamp | LONG | YES |

Additional mandatory parameters based on `type`:

Type | Additional mandatory parameters
------------ |--------------------------------------------------
`LIMIT` | `timeInForce`, `quantity`, `price`
`MARKET` | `quantity`                   
`STOP/TAKE_PROFIT` | `quantity`, `price`, `stopPrice`      
`STOP_MARKET/TAKE_PROFIT_MARKET` | `stopPrice`

- Order with type `STOP`, parameter timeInForce can be sent ( default `GTC`).
- Order with type `TAKE_PROFIT`, parameter `timeInForce` can be sent ( default `GTC`).
- Condition orders will be triggered when:
     - `STOP`, `STOP_MARKET`:
       - BUY: latest price ("MARK_PRICE" or "CONTRACT_PRICE") >= `stopPrice`
       - SELL: latest price ("MARK_PRICE" or "CONTRACT_PRICE") <= `stopPrice`
     - `TAKE_PROFIT`, `TAKE_PROFIT_MARKET`:
       - BUY: latest price ("MARK_PRICE" or "CONTRACT_PRICE") <= `stopPrice`
       - SELL: latest price ("MARK_PRICE" or "CONTRACT_PRICE") >= `stopPrice`
- If `newOrderRespType` is sent as `RESULT` :
     - `MARKET` order: the final FILLED result of the order will be return directly.
     - `LIMIT` order with special `timeInForce`: the final status result of the order(FILLED or EXPIRED) will be returned directly.
- `STOP_MARKET`, `TAKE_PROFIT_MARKET` with `closePosition=true` :
     - Follow the same rules for condition orders.
     - If triggered，close all current long position( if `SELL`) or current short position( if `BUY`).
     - Cannot be used with `quantity` paremeter
     - Cannot be used with `reduceOnly` parameter

**Response:**

```javascript
{
    "orderId": "1745859647816819712",
    "clientOrderId": "6201f5cc-3edf-42ea-a6f5-af799f445c60",
    "symbol": "BTCUSDT",
    "type": "LIMIT",
    "positionSide": "BOTH",
    "timeInForce": "GTC",
    "side": "BUY",
    "isolated": false,
    "origQty": "0.10000000",
    "price": "67000.00000000",
    "reduceOnly": false,
    "executedQty": "0.10000000",
    "status": "NEW",
    "cumQuote": "5136.81520000",
    "avgPrice": "0",
    "stopPrice": "0.00000000",
    "closePosition": false,
    "origType": "LIMIT",
    "updateTime": "1722858688271469674",
    "workingType": "LAST_PRICE"
}
```



#### Place Multiple Orders (TRADE)

```shell
POST /openapi/v1/batchOrders  (HMAC SHA256)
```

**Weight:** 1

**Parameters:**

Name | Type | Mandatory | Description
------------ | ------------ | ------------ | ------------
batchOrders | LIST<JSON> | YES | order list. Max 5 orders
recvWindow | LONG | NO |
timestamp | LONG | YES |

Where batchOrders is the list of order parameters in JSON, recommand to put batchOrders in request body

Name | Type | Mandatory | Description
------------ | ------------ | ------------ | ------------
newClientOrderId | STRING | NO | A unique id among open orders. Automatically generated if not sent. Orders with the same `newClientOrderID` can be accepted only when the previous one is filled, otherwise the order will be rejected.
symbol | STRING | YES |
side | ENUM | YES | `BUY` or `SELL`.
isolated | STRING | NO | `true` or `false`, default `false`.
type | ENUM | YES | `LIMIT`, `MARKET`, `STOP`, `TAKE_PROFIT`, `STOP_MARKET`, `TAKE_PROFIT_MARKET`.
timeInForce | ENUM | NO | `GTC`, `FOK`, `IOC`, `POST_ONLY`, default `GTC`.
quantity | DECIMAL | NO | Cannot be sent with `closePosition=true`(Close-All).
price | DECIMAL | NO |
stopPrice | DECIMAL | NO | Used with `STOP/STOP_MARKET` or `TAKE_PROFIT/TAKE_PROFIT_MARKET` orders.
closePosition | STRING | NO | `true` or `false`. Close-All, used with `STOP_MARKET` or `TAKE_PROFIT_MARKET`.
reduceOnly | STRING | NO | `true` or `false`, default `false`. Cannot be sent with `closePosition=true`.
workingType | ENUM | NO | stopPrice triggered by: `MARK_PRICE` or `LAST_PRICE`, default `LAST_PRICE`.
newOrderRespType | ENUM | NO | `ACK` or `RESULT`, default `ACK`.

- Paremeter rules are same with `New Order`.
- Batch orders are processed concurrently, and the order of matching is not guaranteed.
- The order of returned contents for batch orders is the same as the order of the order list.

**Response:**

```javascript
[
    {
        "orderId": "1745859647816819712",
        "clientOrderId": "6201f5cc-3edf-42ea-a6f5-af799f445c60",
        "symbol": "BTCUSDT",
        "type": "LIMIT",
        "positionSide": "BOTH",
        "timeInForce": "GTC",
        "side": "BUY",
        "isolated": false,
        "origQty": "0.10000000",
        "price": "67000.00000000",
        "reduceOnly": false,
        "executedQty": "0.10000000",
        "status": "NEW",
        "cumQuote": "5136.81520000",
        "avgPrice": "0",
        "stopPrice": "0.00000000",
        "closePosition": false,
        "origType": "LIMIT",
        "updateTime": "1722858688271469674",
        "workingType": "LAST_PRICE"
    }
]
```



#### Query Order (USER_DATA)

```shell
GET /openapi/v1/order  (HMAC SHA256)
```

Check an order's status.

**Weight:** 1

- These orders will not be found:
     - order status is `CANCELED` or `EXPIRED` AND order has NO filled trade AND created time + 3 days < current time
     - order create time + 90 days < current time

**Parameters:**

Name | Type | Mandatory | Description
------------ | ------------ | ------------ | ------------
symbol | STRING | YES |
orderId | LONG | NO |
origClientOrderId | STRING | NO |
recvWindow | LONG | NO |
timestamp | LONG | YES |

Notes:
- Either `orderId` or `origClientOrderId` must be sent. If both parameters are sent, `orderId` takes precedence.

**Response:**

```javascript
{
    "orderId": "1745859647816819712",
    "clientOrderId": "6201f5cc-3edf-42ea-a6f5-af799f445c60",
    "symbol": "BTCUSDT",
    "type": "LIMIT",
    "positionSide": "BOTH",
    "timeInForce": "GTC",
    "side": "BUY",
    "isolated": false,
    "origQty": "0.10000000",
    "price": "67000.00000000",
    "reduceOnly": false,
    "executedQty": "0.10000000",
    "status": "NEW",
    "cumQuote": "5136.81520000",
    "avgPrice": "0",
    "stopPrice": "0.00000000",
    "closePosition": false,
    "origType": "LIMIT",
    "updateTime": "1722858688271469674",
    "workingType": "LAST_PRICE"
}
```



#### Cancel Order (TRADE)

```shell
DELETE /openapi/v1/order  (HMAC SHA256)
```

Cancel an active order.

**Weight:** 1

**Parameters:**

Name | Type | Mandatory | Description
------------ | ------------ | ------------ | ------------
symbol | STRING | YES |
orderId | LONG | NO |
origClientOrderId | STRING | NO |
recvWindow | LONG | NO |
timestamp | LONG | YES |

- Either `orderId` or `origClientOrderId` must be sent. If both parameters are sent, `orderId` takes precedence.

**Response:**

```javascript
{
    "orderId": "1745859647816819712",
    "clientOrderId": "6201f5cc-3edf-42ea-a6f5-af799f445c60",
    "symbol": "BTCUSDT",
    "type": "LIMIT",
    "positionSide": "BOTH",
    "timeInForce": "GTC",
    "side": "BUY",
    "isolated": false,
    "origQty": "0.10000000",
    "price": "67000.00000000",
    "reduceOnly": false,
    "executedQty": "0.10000000",
    "status": "NEW",
    "cumQuote": "5136.81520000",
    "avgPrice": "0",
    "stopPrice": "0.00000000",
    "closePosition": false,
    "origType": "LIMIT",
    "updateTime": "1722858688271469674",
    "workingType": "LAST_PRICE"
}
```



#### Cancel All Open Orders (TRADE)

```shell
DELETE /openapi/v1/allOpenOrders  (HMAC SHA256)
```

**Weight:** 1

**Parameters:**

Name | Type | Mandatory | Description
------------ | ------------ | ------------ | ------------
symbol | STRING | YES |
recvWindow | LONG | NO |
timestamp | LONG | YES |

**Response:**

```javascript
{
    "code": 200,
    "msg": "The operation of cancel all open order is done."
}
```



#### Cancel Multiple Orders (TRADE)

```shell
DELETE /openapi/v1/batchOrders  (HMAC SHA256)
```

**Weight:** 1

**Parameters:**

Name | Type | Mandatory | Description
------------ | ------------ | ------------ | ------------
symbol | STRING | YES |
orderIdList | `LIST<LONG>` | NO | max length 10, e.g. [1234567,2345678].
origClientOrderIdList | `LIST<STRING>` | NO | max length 10, e.g. ["my_id_1","my_id_2"], encode the double quotes. No space after comma.
recvWindow | LONG | NO |
timestamp | LONG | YES |

- Either `orderIdList` or `origClientOrderIdList` must be sent. If both parameters are sent, `orderIdList` takes precedence.

**Response:**

```javascript
[
  {
    "orderId": "1745859647816819712",
    "clientOrderId": "6201f5cc-3edf-42ea-a6f5-af799f445c60",
    "symbol": "BTCUSDT",
    "type": "LIMIT",
    "positionSide": "BOTH",
    "timeInForce": "GTC",
    "side": "BUY",
    "isolated": false,
    "origQty": "0.10000000",
    "price": "67000.00000000",
    "reduceOnly": false,
    "executedQty": "0.10000000",
    "status": "NEW",
    "cumQuote": "5136.81520000",
    "avgPrice": "0",
    "stopPrice": "0.00000000",
    "closePosition": false,
    "origType": "LIMIT",
    "updateTime": "1722858688271469674",
    "workingType": "LAST_PRICE"
  }
  {
    "code": -1136,
    "msg": "Unknown order sent."
  }
]
```



#### New Futures Account Transfer (USER_DATA)

```shell
POST /openapi/v1/transfer (HMAC SHA256)
```

Execute transfer between spot account and futures account.

**Weight:** 1

**Parameters:**

Name | Type | Mandatory | Description
------------ | ------------ | ------------ | ------------
asset | STRING | YES | The asset being transferred, e.g., USDT.
amount | DECIMAL | YES | The amount to be transferred.
type | INT | YES | 1: transfer from spot account to USDT-Ⓜ futures account, 2: transfer from USDT-Ⓜ futures account to spot account.
recvWindow | LONG | NO |
timestamp | LONG | YES |

- You need to open Enable Futures permission for the API Key which requests this endpoint.
  
**Response:**

```javascript
{
    "tranId": 100000001    //transaction id
}
```



#### Query Current Open Order (USER_DATA)

```shell
GET /openapi/v1/openOrder (HMAC SHA256)
```

**Weight:** 1

**Parameters:**

Name | Type | Mandatory | Description
------------ | ------------ | ------------ | ------------
symbol | STRING | YES |
orderId | LONG | NO |
origClientOrderId | STRING | NO |
recvWindow | LONG | NO |
timestamp | LONG | YES |

- Either `orderId` or `origClientOrderId` must be sent. If both parameters are sent, `orderId` takes precedence.
- If the queried order has been filled or cancelled, the error message "Order does not exist" will be returned.
  
**Response:**

```javascript
{
    "orderId": "1745859647816819712",
    "clientOrderId": "6201f5cc-3edf-42ea-a6f5-af799f445c60",
    "symbol": "BTCUSDT",
    "type": "LIMIT",
    "positionSide": "BOTH",
    "timeInForce": "GTC",
    "side": "BUY",
    "isolated": false,
    "origQty": "0.10000000",
    "price": "67000.00000000",
    "reduceOnly": false,
    "executedQty": "0.10000000",
    "status": "NEW",
    "cumQuote": "5136.81520000",
    "avgPrice": "0",
    "stopPrice": "0.00000000",
    "closePosition": false,
    "origType": "LIMIT",
    "updateTime": "1722858688271469674",
    "workingType": "LAST_PRICE"
}
```



#### Current All Open Orders (USER_DATA)

```shell
GET /openapi/v1/openOrders (HMAC SHA256)
```

Get all open orders on a symbol. Careful when accessing this with no symbol.

**Weight:** 1 for a single symbol; 40 when the symbol parameter is omitted

**Parameters:**

Name | Type | Mandatory | Description
------------ | ------------ | ------------ | ------------
symbol | STRING | NO |
recvWindow | LONG | NO |
timestamp | LONG | YES |

- If the symbol is not sent, orders for all symbols will be returned in an array.
  
**Response:**

```javascript
[
  {
    "orderId": "1745859647816819712",
    "clientOrderId": "6201f5cc-3edf-42ea-a6f5-af799f445c60",
    "symbol": "BTCUSDT",
    "type": "LIMIT",
    "positionSide": "BOTH",
    "timeInForce": "GTC",
    "side": "BUY",
    "isolated": false,
    "origQty": "0.10000000",
    "price": "67000.00000000",
    "reduceOnly": false,
    "executedQty": "0.10000000",
    "status": "NEW",
    "cumQuote": "5136.81520000",
    "avgPrice": "0",
    "stopPrice": "0.00000000",
    "closePosition": false,
    "origType": "LIMIT",
    "updateTime": "1722858688271469674",
    "workingType": "LAST_PRICE"
  }
]
```



#### All Orders (USER_DATA)

```shell
GET /openapi/v1/allOrders (HMAC SHA256)
```

Get all account orders; active, canceled, or filled.

- These orders will not be found:
     - order status is `CANCELED` or `EXPIRED` AND order has NO filled trade AND created time + 3 days < current time
     - order create time + 90 days < current time

**Weight:** 5

**Parameters:**

Name | Type | Mandatory | Description
------------ | ------------ | ------------ | ------------
symbol | STRING | YES |
orderId | LONG | NO |
startTime | LONG | NO |
endTime | LONG | NO |
limit | INT | NO | Default 500; max 1000.
recvWindow | LONG | NO |
timestamp | LONG | YES |

Notes:
- If `orderId` is set, it will get orders >= that `orderId`. Otherwise most recent orders are returned.
- The query time period must be less then 7 days( default as the recent 7 days).

**Response:**

```javascript
[
  {
    "orderId": "1745859647816819712",
    "clientOrderId": "6201f5cc-3edf-42ea-a6f5-af799f445c60",
    "symbol": "BTCUSDT",
    "type": "LIMIT",
    "positionSide": "BOTH",
    "timeInForce": "GTC",
    "side": "BUY",
    "isolated": false,
    "origQty": "0.10000000",
    "price": "67000.00000000",
    "reduceOnly": false,
    "executedQty": "0.10000000",
    "status": "NEW",
    "cumQuote": "5136.81520000",
    "avgPrice": "0",
    "stopPrice": "0.00000000",
    "closePosition": false,
    "origType": "LIMIT",
    "updateTime": "1722858688271469674",
    "workingType": "LAST_PRICE"
  }
]
```



#### Current Leverage (USER_DATA)

```shell
GET /openapi/v1/leverage (HMAC SHA256)
```

Get user's current leverage information.

**Weight:** 1

**Parameters:**

Name | Type | Mandatory | Description
------------ | ------------ | ------------ | ------------
recvWindow | LONG | NO |
timestamp | LONG | YES |
  
**Response:**

```javascript
[
    {
        "symbol": "ETCUSDT",
        "crossLeverage": 6,
        "isolatedLeverage": 6
    },
    {
        "symbol": "AXSUSDT",
        "crossLeverage": 2,
        "isolatedLeverage": 2
    },
    {
        "symbol": "BCHUSDT",
        "crossLeverage": 3,
        "isolatedLeverage": 3
    },
    {
        "symbol": "TESUSDT",
        "crossLeverage": 6,
        "isolatedLeverage": 6
    },
    {
        "symbol": "LINKUSDT",
        "crossLeverage": 3,
        "isolatedLeverage": 3
    },
    {
        "symbol": "BTCUSDT",
        "crossLeverage": 20,
        "isolatedLeverage": 5
    }
]
```


#### Change Initial Leverage (USER_DATA)

```shell
POST /openapi/v1/leverage (HMAC SHA256)
```

Change user's initial leverage of specific symbol market.

**Weight:** 1

**Parameters:**

Name | Type | Mandatory | Description
------------ | ------------ | ------------ | ------------
symbol | STRING | YES |
leverage | INT | YES | target initial leverage: int from 1 to 125.
isolated | STRING | NO | `true` or `false`, default `false`.
recvWindow | LONG | NO |
timestamp | LONG | YES |
  
**Response:**

```javascript
{
    "leverage": 21,
    "isolated": "true",
    "symbol": "BTCUSDT"
}
```



#### Modify Isolated Position Margin (TRADE)

```shell
POST /openapi/v1/positionMargin (HMAC SHA256)
```

**Weight:** 1

**Parameters:**

Name | Type | Mandatory | Description
------------ | ------------ | ------------ | ------------
symbol | STRING | YES |
amount | DECIMAL | YES |
type | INT | YES | 	1: Add position margin，2: Reduce position margin.
recvWindow | LONG | NO |
timestamp | LONG | YES |

- Only for isolated symbol
  
**Response:**

```javascript
{
    "amount": "100.0",
    "type": 1
}
```



#### User Commission Rate (USER_DATA)

```shell
GET /openapi/v1/commissionRate (HMAC SHA256)
```

**Weight:** 20

**Parameters:**

Name | Type | Mandatory | Description
------------ | ------------ | ------------ | ------------
symbol | STRING | YES |
recvWindow | LONG | NO |
timestamp | LONG | YES |
  
**Response:**

```javascript
{
    "symbol": "BTCUSDT",
    "makerCommissionRate": "0.0002",  // 0.02%
    "takerCommissionRate": "0.0004"   // 0.04%
}
```



#### Futures Account Balance (USER_DATA)

```shell
GET /openapi/v1/balance (HMAC SHA256)
```

**Weight:** 5

**Parameters:**

Name | Type | Mandatory | Description
------------ | ------------ | ------------ | ------------
recvWindow | LONG | NO |
timestamp | LONG | YES |
  
**Response:**

```javascript
[
    {
        "tokenId": "USDT",
        "balance": "1193.9402"
    }
]
```



#### Account Information (USER_DATA)

```shell
GET /openapi/v1/account (HMAC SHA256)
```

Get current account information.

**Weight:** 5

**Parameters:**

Name | Type | Mandatory | Description
------------ | ------------ | ------------ | ------------
recvWindow | LONG | NO |
timestamp | LONG | YES |
  
**Response:**

```javascript
{
    "assets": [
        {
            "tokenId": "USDT",
            "balance": "1193.9402",
            "version": 10
        }
    ],
    "positions": [  // positions of all symbols in the market are returned
        {
            "userId": "1697173266836457216",
            "symbolId": "BTCUSDT",
            "baseToken": "BTC",
            "quoteToken": "USDT",
            "settleToken": "USDT",
            "avgEntryPrice": "6059.8",
            "quantity": "5",
            "curTermRealisedPnl": "0",
            "isolated": false,
            "isolatedMargin": "0",
            "positionSide": "BOTH",
            "markPrice": "65215.971",
            "unrealizedProfit": null,
            "leverage": null,
            "liquidationPrice": null
        }
    ]
}
```



### Market Data endpoints

#### Order book

```shell
GET /openapi/quote/v1/depth
```

**Weight:**

Adjusted based on the limit:

Limit | Weight
------------ | ------------
5, 10, 20, 50, 100 | 1
200 | 5

**Parameters:**

Name | Type | Mandatory | Description
------------ | ------------ | ------------ | ------------
symbol | STRING | YES |
limit | INT | NO | Default 100; max 200.

**Caution:** setting limit=0 can return 200 records.

**Response:**

[PRICE, QTY]

```javascript
{
  "lastUpdateId": 1027024,
  "bids": [
    [
      "4.90000000",   // PRICE
      "331.00000000"  // QTY
    ],
    [
      "4.00000000",
      "431.00000000"
    ]
  ],
  "asks": [
    [
      "4.00000200",  // PRICE
      "12.00000000"  // QTY
    ],
    [
      "5.10000000",
      "28.00000000"
    ]
  ]
}
```



#### Recent trades list

```shell
GET /openapi/quote/v1/trades
```

Get recent trades (up to last 60).

**Weight:** 1

**Parameters:**

Name | Type | Mandatory | Description
------------ | ------------ | ------------ | ------------
symbol | STRING | YES |EXP: BTCUSDT
limit | INT | NO | Default 500; max 1000. if limit <=0 or > 1000 then return 1000

**Response:**

```javascript
[
  {
    "id": 28457,
    "price": "4.00000100",
    "qty": "12.00000000",
    "quoteQty": "48.000012",
    "time": 1499865549590,
    "isBuyerMaker": true
  }
]
```



#### Old Trades Lookup 

```shell
GET /openapi/quote/v1/historicalTrades
```

Get older market historical trades.
Only supports returning data from the last three months.

**Weight:** 20

**Parameters:**

Name | Type | Mandatory | Description
------------ | ------------ | ------------ | ------------
symbol | STRING | YES |EXP: BTCUSDT
limit | INT | NO | Default 500; max 1000. if limit <=0 or > 1000 then return 1000.
fromId | LONG | NO | TradeId to fetch from. Default gets most recent trades.

**Response:**

```javascript
[
  {
    "id": 28457,
    "price": "4.00000100",
    "qty": "12.00000000",
    "quoteQty": "48.000012",
    "time": 1499865549590,
    "isBuyerMaker": true
  }
]
```



#### Kline/Candlestick data

```shell
GET /openapi/quote/v1/klines
```

Kline/candlestick bars for a symbol.
Klines are uniquely identified by their open time.

**Weight:** 1

**Parameters:**

Name | Type | Mandatory | Description
------------ | ------------ | ------------ | ------------
symbol | STRING | YES |EXP: BTCUSDT
interval | ENUM | YES |
startTime | LONG | NO |
endTime | LONG | NO |
limit | INT | NO | Default 500; max 1000.

* If startTime and endTime are not sent, the most recent klines are returned.

**Response:**

```javascript
[
  [
    1499040000000,      // Open time
    "0.01634790",       // Open
    "0.80000000",       // High
    "0.01575800",       // Low
    "0.01577100",       // Close
    "148976.11427815",  // Volume
    1499644799999,      // Close time
    "2434.19055334",    // Quote asset volume
    308,                // Number of trades
    "1756.87402397",    // Taker buy base asset volume
    "28.46694368"       // Taker buy quote asset volume
  ]
]
```



#### Mark Price Kline/Candlestick Data

```shell
GET /openapi/quote/v1/markPriceKlines
```

Kline/candlestick bars for the mark price of a symbol.
Klines are uniquely identified by their open time.

**Weight:** 1

**Parameters:**

Name | Type | Mandatory | Description
------------ | ------------ | ------------ | ------------
symbol | STRING | YES |EXP: BTCUSDT
interval | ENUM | YES |
startTime | LONG | NO |
endTime | LONG | NO |
limit | INT | NO | Default 500; max 1000.

* If startTime and endTime are not sent, the most recent klines are returned.

**Response:**

```javascript
[
  [
    1499040000000,      // Open time
    "0.01634790",       // Open
    "0.80000000",       // High
    "0.01575800",       // Low
    "0.01577100",       // Close
    1499644799999       // Close time
  ]
]
```



#### Current average price

```shell
GET /openapi/quote/v1/avgPrice
```

Current average price for a symbol.

**Weight:** 1

**Parameters:**

| Name   | Type   | Mandatory | Description                                           |
| ------ | ------ | --------- | ----------------------------------------------------- |
| symbol | STRING | YES       | symbol is not case sensitive, e.g. BTCUSDT or btcusdt |


**Response:**

```javascript
{
  "mins": 5,
  "price": "9.35751834"
}
```

#### 24hr ticker price change statistics

```shell
GET /openapi/quote/v1/ticker/24hr
```

24 hour price change statistics. **Careful** when accessing this with no symbol.

**Weight:**

| Parameter | Symbols Provided            | Weight |
| --------- | --------------------------- | ------ |
| symbol    | 1                           | 1      |
|           | symbol parameter is omitted | 40     |
| symbols   | 1-20                        | 1      |
|           | 21-100                      | 20     |
|           | 101 or more                 | 40     |
|           | symbol parameter is omitted | 40     |

**Parameters:**

Name | Type | Mandatory | Description
------------ | ------------ | ------------ | ------------
symbol | STRING | NO |Example: BTCUSDT
symbols | STRING | NO |

* Parameter symbol and symbols cannot be used in combination.If neither parameter is sent, tickers for all symbols will be returned in an array.Examples of accepted format for the symbols parameter: ["BTCUSDT","BNBUSDT"] and not case sensitive

**Response:**

```javascript
{
  "symbol": "BNBBTC",
  "priceChange": "-94.99999800",
  "priceChangePercent": "-95.960",
  "weightedAvgPrice": "0.29628482",
  "prevClosePrice": "0.10002000",
  "lastPrice": "4.00000200",
  "lastQty": "200.00000000",
  "bidPrice": "4.00000000",
  "bidQty": "100.00000000",
  "askPrice": "4.00000200",
  "askQty": "100.00000000",
  "openPrice": "99.00000000",
  "highPrice": "100.00000000",
  "lowPrice": "0.10000000",
  "volume": "8913.30000000",
  "quoteVolume": "15.30000000",
  "openTime": 1499783499040,
  "closeTime": 1499869899040,
  "firstId": 28385,   // first trade id
  "lastId": 28460,    // Last tradeId
  "count": 76         // Trade count
}

```

OR

```javascript
[
  {
    "symbol": "BNBBTC",
    "priceChange": "-94.99999800",
    "priceChangePercent": "-95.960",
    "weightedAvgPrice": "0.29628482",
    "prevClosePrice": "0.10002000",
    "lastPrice": "4.00000200",
    "lastQty": "200.00000000",
    "bidPrice": "4.00000000",
    "bidQty": "100.00000000",
    "askPrice": "4.00000200",
    "askQty": "100.00000000",
    "openPrice": "99.00000000",
    "highPrice": "100.00000000",
    "lowPrice": "0.10000000",
    "volume": "8913.30000000",
    "quoteVolume": "15.30000000",
    "openTime": 1499783499040,
    "closeTime": 1499869899040,
    "firstId": 28385,   // First tradeId
    "lastId": 28460,    // Last tradeId
    "count": 76         // Trade count
  }
]
```



#### Symbol price ticker

```shell
GET /openapi/quote/v1/ticker/price
```

Latest price for a symbol or symbols.

**Weight:**

| Parameter | Symbols Provided            | Weight |
| --------- | --------------------------- | ------ |
| symbol    | 1                           | 1      |
|           | symbol parameter is omitted | 2      |
| symbols   | Any                         | 2      |

**Parameters:**

Name | Type | Mandatory | Description
------------ | ------------ | ------------ | ------------
symbol | STRING | NO |Example: BTCUSDT
symbols | STRING | NO |

* Parameter symbol and symbols cannot be used in combination.If neither parameter is sent, prices for all symbols will be returned in an array.Examples of accepted format for the symbols parameter: ["BTCUSDT","BNBUSDT"] and not case sensitive

**Response:**

```javascript
{
  "symbol": "LTCBTC",
  "price": "4.00000200"
}
```

OR

```javascript
[
  {
    "symbol": "LTCBTC",
    "price": "4.00000200"
  },
  {
    "symbol": "ETHBTC",
    "price": "0.07946600"
  }
]
```



#### Symbol order book ticker

```shell
GET /openapi/quote/v1/ticker/bookTicker
```

Best price/qty on the order book for a symbol or symbols.

**Weight:**

| Parameter | Symbols Provided            | Weight |
| --------- | --------------------------- | ------ |
| symbol    | 1                           | 1      |
|           | symbol parameter is omitted | 2      |
| symbols   | Any                         | 2      |

**Parameters:**

Name | Type | Mandatory | Description
------------ | ------------ | ------------ | ------------
symbol | STRING | NO |
symbols | STRING | NO |

* Parameter symbol and symbols cannot be used in combination.If neither parameter is sent, bookTickers for all symbols will be returned in an array.Examples of accepted format for the symbols parameter: ["BTCUSDT","BNBUSDT"] and not case sensitive

**Response:**

```javascript
{
  "symbol": "LTCBTC",
  "bidPrice": "4.00000000",
  "bidQty": "431.00000000",
  "askPrice": "4.00000200",
  "askQty": "9.00000000"
}
```

OR

```javascript
[
  {
    "symbol": "LTCBTC",
    "bidPrice": "4.00000000",
    "bidQty": "431.00000000",
    "askPrice": "4.00000200",
    "askQty": "9.00000000"
  },
  {
    "symbol": "ETHBTC",
    "bidPrice": "0.07946700",
    "bidQty": "9.00000000",
    "askPrice": "100000.00000000",
    "askQty": "1000.00000000"
  }
]
```


 
#### Crypto asset trading pairs

```shell
GET /openapi/v1/pairs
```
A summary on crypto asset trading pairs available on the exchange

**Weight:** 1

**Parameters:**

None

**Response:**

```javascript
[
    {
        "symbol": "ETCUSDT",
        "quoteToken": "USDT",
        "baseToken": "ETC"
    },
    {
        "symbol": "AXSUSDT",
        "quoteToken": "USDT",
        "baseToken": "AXS"
    },
    {
        "symbol": "BCHUSDT",
        "quoteToken": "USDT",
        "baseToken": "BCH"
    },
    {
        "symbol": "TESUSDT",
        "quoteToken": "USDT",
        "baseToken": "TES"
    },
    {
        "symbol": "BTCUSDT",
        "quoteToken": "USDT",
        "baseToken": "BTC"
    }
]
```

### User data stream endpoints

Specifics on how user data streams work is in another document(user-data-stream.md).



#### Start user data stream (USER_STREAM)

```shell
POST /openapi/v1/userDataStream
```

Start a new user data stream. The stream will close after 60 minutes unless a keep alive is sent.

**Weight:** 1

**Parameters:**

None

**Response:**

```javascript
{
  "listenKey": "xDqtskqOciCzRashthgjTHBcymasBBShEEzPiXgOGEujviYWCuyYwcPDVPeezJOT"
}
```



#### Keep alive user data stream (USER_STREAM)

```shell
PUT /openapi/v1/userDataStream
```

Keep alive a user data stream to prevent a time out. User data streams will close after 60 minutes. It's recommended to send a ping about every 30 minutes.

**Weight:** 1

**Parameters:**

Name | Type | Mandatory | Description
------------ | ------------ | ------------ | ------------
listenKey | STRING | YES |

**Response:**

```javascript
{}
```



#### Close user data stream (USER_STREAM)

```shell
DELETE /openapi/v1/userDataStream
```

Close out a user data stream.

**Weight:** 1

**Parameters:**

Name | Type | Mandatory | Description
------------ | ------------ | ------------ | ------------
listenKey | STRING | YES |

**Response:**

```javascript
{}
```



### Sub-account endpoints

### Query Sub-account List (For Master Account)

Applies to master accounts only.

```shell
GET /openapi/v1/sub-account/list
```

**Weight:** 1

**Parameters:**

Name       | Type  | Mandatory | Description
-----------------|--------|-----------|--------------------------------------------------------------------------------------
email      | STRING | NO    | <a href="#request-parameters">Sub-account email</a>
page    | INT | NO | Current page, default value: 1
limit    | INT | NO | Quantity per page, default value 10, maximum `200`
recvWindow | LONG  | NO    | This value cannot be greater than `60000`
timestamp     | LONG  | YES    | A point in time for which transfers are being queried.


**Response:**
```json
{
  "subAccounts": [
    {
      "createTime": "1689744671462",
      "email": "testsub@gmail.com",
      "isFreeze": false
    },
    {
      "createTime": "1689744700710",
      "email": "testsub2@gmail.com",
      "isFreeze": false
    }
  ],
 "total": 2
}
```

### Create a Virtual Sub-account(For Master Account)

This interface currently supports the creation of virtual sub-accounts (maximum 30).

```shell
POST /openapi/v1/sub-account/create
```

**Weight:** 1

**Parameters:**

Name       | Type  | Mandatory | Description
-----------------|--------|-----------|--------------------------------------------------------------------------------------
accountName      | STRING | YES       | <a href="#request-parameters">Sub-account email</a>
recvWindow | LONG  | NO        | This value cannot be greater than `60000`
timestamp     | LONG  | YES       | A point in time for which transfers are being queried.


**Response:**
```json
{
  "email": "testsub@gmail.com",
  "createTime": 1689744700710,
  "isFreeze": false
}
```


### Query Sub-account Assets (For Master Account)

Query detailed balance information of a sub-account via the master account (applies to master accounts only).

```shell
GET /openapi/v1/sub-account/asset
```

**Weight:** 1

**Parameters:**

Name       | Type  | Mandatory | Description
-----------------|--------|-----------|--------------------------------------------------------------------------------------
email      | STRING | YES       | <a href="#request-parameters">Sub-account email</a>
recvWindow | LONG  | NO        | This value cannot be greater than `60000`
timestamp     | LONG  | YES       | A point in time for which transfers are being queried.


**Response:**
```json
{
  "balances": [
    {
      "asset": "BTC",
      "free": "0.1",
      "locked": "0"
    },
    {
      "asset": "ETH",
      "free": "0.1",
      "locked": "0"
    }
  ]
}
```



### Universal Transfer (For Master Account)

Master account can initiate a transfer from any of its sub-accounts to the master account, or from the master account to any sub-account.

```shell
POST /openapi/v1/sub-account/transfer/universal-transfer
```

**Weight:** 1

**Parameters:**

Name       | Type  | Mandatory | Description
-----------------|--------|-----------|--------------------------------------------------------------------------------------
fromEmail      | STRING | NO        | 
toEmail      | STRING | NO        | 
clientTranId      | STRING | NO        | Must be unique
asset      | STRING | YES       | 
amount      | DECIMAL | YES        | 
recvWindow | LONG  | NO        | This value cannot be greater than `60000`
timestamp     | LONG  | YES       | A point in time for which transfers are being queried.

- Transfer from master account by default if fromEmail is not sent.
- Transfer to master account by default if toEmail is not sent.
- Specify at least one of fromEmail and toEmail.

**Response:**
```json
{
  "clientTransferId": "1487573639841995271",
  "result": true//true:success,false:failed
}
```

### Transfer to Master (For Sub-account)

Sub-account can initiate a transfer from itself to the master account.

```shell
POST /openapi/v1/sub-account/transfer/sub-to-master
```

**Weight:** 1

**Parameters:**

Name       | Type  | Mandatory | Description
-----------------|--------|-----------|--------------------------------------------------------------------------------------
asset      | STRING | YES       |
amount      | DECIMAL | YES        |
clientTranId      | STRING | NO        | Must be unique
recvWindow | LONG  | NO        | This value cannot be greater than `60000`
timestamp     | LONG  | YES       | A point in time for which transfers are being queried.


**Response:**
```json
{
  "clientTransferId": "1487573639841995271",
  "result": true//true:success,false:failed
}
```

### Query Universal Transfer History (For Master Account)

Applies to master accounts only.
If startTime and endTime are not sent, this will return records of the last 30 days by default.

```shell
GET /openapi/v1/sub-account/transfer/universal-transfer-history
```

**Weight:** 1

**Parameters:**

Name       | Type  | Mandatory | Description
-----------------|--------|-----------|--------------------------------------------------------------------------------------
fromEmail      | STRING | NO        |
toEmail      | STRING | NO        |
clientTranId      | STRING | NO        | 
tokenId      | STRING | NO        | 
startTime      | LONG | NO        | Millisecond timestamp
endTime      | LONG | NO        | Millisecond timestamp,Data excluding the endTime.
page      | INT | NO        | Current page, default value: 1
limit      | INT | NO        | Quantity per page, default value `500`, maximum `500`
recvWindow | LONG  | NO        | This value cannot be greater than `60000`
timestamp     | LONG  | YES       | A point in time for which transfers are being queried.


- fromEmail and toEmail cannot be sent at the same time.
- Return fromEmail equal master account email by default.
- The query time period must be less than 30 days. 
- If startTime and endTime not sent, return records of the last 30 days by default.

**Response:**
```json
{
  "result": [
    {
      "clientTranId": "1",
      "fromEmail": "testsub@gmail",
      "toEmail": "testsub1@gmail",
      "asset": "BTC",
      "amount": "0.1",
      "createdAt": 1689744700710,
      "status": "success"//success,pending,failed
    }
  ],
  "total": 1
}
```


### Sub-account Transfer History (For Sub-account)

Applies to sub-accounts only.
If startTime and endTime are not sent, this will return records of the last 30 days by default.

```shell
GET /openapi/v1/sub-account/transfer/sub-history
```

**Weight:** 1

**Parameters:**

Name       | Type   | Mandatory | Description
-----------------|--------|-----------|--------------------------------------------------------------------------------------
asset      | STRING | NO        |
type      | INT | NO        | 1: transfer in, 2: transfer out. If the type parameter is not provided or provided incorrectly, the data returned will be for transfer out.
startTime      | LONG   | NO        | Millisecond timestamp
endTime      | LONG   | NO        | Millisecond timestamp,Data excluding the endTime.
page      | INT    | NO        | Current page, default value: 1
limit      | INT | NO        | Quantity per page, default value `500`, maximum `500`
recvWindow | LONG   | NO        | This value cannot be greater than `60000`
timestamp     | LONG   | YES       | A point in time for which transfers are being queried.

- If type is not sent, the records of type 2: transfer out will be returned by default.
- If startTime and endTime are not sent, the recent 30-day data will be returned.

**Response:**
```json
{
  "result": [
    {
      "clientTranId": "1",
      "fromEmail": "testsub@gmail",
      "toEmail": "testsub1@gmail",
      "asset": "BTC",
      "amount": "0.1",
      "createdAt": 1689744700710,
      "status": "success"//success,pending,failed
    }
  ],
  "total": 1
}
```


### Get IP Restriction for a Sub-account API Key (For Master Account)

Query detailed IPs for a sub-account API key.

```shell
GET /openapi/v1/sub-account/apikey/ip-restriction
```

**Weight:** 1

**Parameters:**

Name       | Type   | Mandatory | Description
-----------------|--------|-----------|--------------------------------------------------------------------------------------
apikey      | STRING | YES        | 
email      | STRING | YES        | 	<a href="#request-parameters">Sub-account email</a>
recvWindow | LONG   | NO        | This value cannot be greater than `60000`
timestamp     | LONG   | YES       | A point in time for which transfers are being queried.


**Response:**
```json
{
  "apikey": "k5V49ldtn4tszj6W3hystegdfvmGbqDzjmkCtpTvC0G74WhK7yd4rfCTo4lShf",
  "ipList": [
    "8.8.8.8"
  ],
  "ipRestrict": true,
  "role": "2,3,4,5,6",//0:READ_ONLY, 2:TRADE_ONLY, 3:CONVERT_ONLY, 4:CRYPTO_WALLET_ONLY, 5:FIAT_ONLY, 6:ACCOUNT_ONLY
  "updateTime": 1689744700710
}
```

###  Add IP Restriction for Sub-Account API key (For Master Account)

```shell
POST /openapi/v1/sub-account/apikey/add-ip-restriction
```

**Weight:** 1

**Parameters:**

Name       | Type   | Mandatory | Description
-----------------|--------|-----------|--------------------------------------------------------------------------------------
apikey      | STRING | YES       |
email      | STRING | YES       | 	<a href="#request-parameters">Sub-account email</a>
ipAddress      | STRING | NO        | 	Can be added in batches, separated by commas
ipRestriction      | STRING | YES       | 	IP Restriction status. 2 = IP Unrestricted. 1 = Restrict access to trusted IPs only.
recvWindow | LONG   | NO        | This value cannot be greater than `60000`
timestamp     | LONG   | YES       | A point in time for which transfers are being queried.


**Response:**
```json
{
  "apikey": "k5V49ldtn4tszj6W3hystegdfvmGbqDzjmkCtpTvC0G74WhK7yd4rfCTo4lShf",
  "ipList": [
    "8.8.8.8"
  ],
  "ipRestrict": true,
  "role": "2,3,4,5,6",//0:READ_ONLY, 2:TRADE_ONLY, 3:CONVERT_ONLY, 4:CRYPTO_WALLET_ONLY, 5:FIAT_ONLY, 6:ACCOUNT_ONLY
  "updateTime": 1689744700710
}
```

###  Delete IP List For a Sub-account API Key (For Master Account)

```shell
POST /openapi/v1/sub-account/apikey/delete-ip-restriction
```

**Weight:** 1

**Parameters:**

Name       | Type   | Mandatory | Description
-----------------|--------|-----------|--------------------------------------------------------------------------------------
apikey      | STRING | YES       |
email      | STRING | YES       | 	<a href="#request-parameters">Sub-account email</a>
ipAddress      | STRING | YES       | 	Can be added in batches, separated by commas
recvWindow | LONG   | NO        | This value cannot be greater than `60000`
timestamp     | LONG   | YES       | A point in time for which transfers are being queried.


**Response:**
```json
{
  "apikey": "k5V49ldtn4tszj6W3hystegdfvmGbqDzjmkCtpTvC0G74WhK7yd4rfCTo4lShf",
  "ipList": [
    "8.8.8.8"
  ],
  "ipRestrict": true,
  "role": "2,3,4,5,6",//0:READ_ONLY, 2:TRADE_ONLY, 3:CONVERT_ONLY, 4:CRYPTO_WALLET_ONLY, 5:FIAT_ONLY, 6:ACCOUNT_ONLY
  "updateTime": 1689744700710
}
```



### Note

### Request Parameters

- Email address should be encoded. e.g. test@gmail.com should be encoded into test%40gmail.com
