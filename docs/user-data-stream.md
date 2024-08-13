---
permalink: /user-data-stream/
title: "User-Data-Stream"
nav: sidebar/user-data-stream.html
layout: default
---



# User Data Streams for Coins Futures (2024-08-31)

## General WSS information

* The base endpoint is: **https://fapi.coins.xyz**
* A User Data Stream `listenKey` is valid for 60 minutes after creation.
* Doing a `PUT` on a `listenKey` will extend its validity for 60 minutes.
* Doing a `DELETE` on a `listenKey` will close the stream and invalidate the listenKey.
* Doing a `POST` on an account with an active `listenKey` will return the currently active `listenKey` and extend its validity for 60 minutes.
* The base websocket endpoint is: **wss://wsapi.pro.coins.ph**
* User Data Streams are accessed at **/openapi/ws/\<listenKey\>**
* A single connection to api endpoint is only valid for 24 hours; expect to be disconnected at the 24 hour mark

## API Endpoints

### Create a listenKey

```shell
POST /openapi/v1/userDataStream
```

Start a new user data stream. The stream will close after 60 minutes unless a keepalive is sent.that listenKey will be returned and its validity will be extended for 60 minutes.

**Weight:** 1

**Parameters:**

None

**Response:**

```javascript
{
  "listenKey": "1A9LWJjuMwKWYP4QQPw34GRm8gz3x5AephXSuqcDef1RnzoBVhEeGE963CoS1Sgj"
}
```

### Ping/Keep-alive a listenKey

```shell
PUT /openapi/v1/userDataStream
```

Keepalive a user data stream to prevent a time out. User data streams will close after 60 minutes. It's recommended to send a ping about every 30 minutes.

**Weight:** 1

**Parameters:**

Name | Type | Mandatory | Description
------------ | ------------ | ------------ | ------------
listenKey | STRING | YES |

**Response:**

```javascript
{}
```

### Close a listenKey

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

## Web Socket Payloads

### Balance and Position Update

Event type is `ACCOUNT_UPDATE`.

- When balance or position get updated, this event will be pushed.
  - `ACCOUNT_UPDATE` will be pushed only when update happens on user's account, including changes on balances, positions, or margin type.
  - Unfilled orders or cancelled orders will not make the event `ACCOUNT_UPDATE` pushed, since there's no change on positions.
  - "position" in `ACCOUNT_UPDATE`: Only symbols of changed positions will be pushed.
- When "FUNDING FEE" changes to the user's balance, the event will be pushed with the brief message:
  - When "FUNDING FEE" occurs in a crossed position, `ACCOUNT_UPDATE` will be pushed with only the balance `B`(including the "FUNDING FEE" asset only), without any position `P` message.
  - When "FUNDING FEE" occurs in an isolated position, `ACCOUNT_UPDATE` will be pushed with only the balance `B`(including the "FUNDING FEE" asset only) and the relative position message `P` (including the isolated position on which the "FUNDING FEE" occurs only, without any other position message).
- The field "m" represents the reason type for the event.
- The field "bc" represents the balance change except for PnL and commission.

**Payload:**

```javascript
{
    "e": "ACCOUNT_UPDATE",                     // Event Type
    "E": 1719213612440,                        // Event Time
    "T": 1719213612248210972,                  // Transaction
    "a": {                                     // Update Data
        "m": "ORDER",                          // Event reason type
        "B": [                                 // Balances
            {
                "a": "USDT",                   // Asset
                "wb": 1193.936750000000000000, // Wallet Balance
                "cw": 1193.936750000000000000, // Cross Wallet Balance
                "bc": -0.003450000000000000    // Balance Change except PnL and Commission
            }
        ],
        "P": [
            {
                "s": "BTCUSDT",                 // Symbol
                "pa": 5.000100000000000000,     // Position Amount
                "ep": 6061.058778824424000000,  // Entry Price
                "bep": 6061.058778824424000000, // breakeven price
                "cr": 0E-18,                    // (Pre-fee) Accumulated Realized
                "up": 0,                        // Unrealized PnL
                "mt": "CROSSED",                // Margin Type
                "iw": 0E-18,                    // Isolated Wallet (if isolated position)
                "ps": "BOTH"                    // Position Side
            }
        ]
    }
}
```


### Order Update

When new order created, order status changed will push such event. event type is `ORDER_TRADE_UPDATE`.

- Side
  - BUY
  - SELL
 
- Order Type
  - LIMIT
  - MARKET_OF_BASE
  - STOP_LOSS_LIMIT
  - TAKE_PROFIT_LIMIT
  - STOP_LOSS
  - TAKE_PROFIT

- Execution Type
  - NEW
  - CANCELED
  - REJECTED
  - TRADE
  - EXPIRED

- Order Status
  - NEW
  - PARTIALLY_FILLED
  - FILLED
  - PARTIALLY_CANCELED
  - CANCELED
  - REJECTED
  - EXPIRED
 
- Time in force
  - GTC
  - FOK
  - IOC
  - POST_ONLY
 
- Working Type
  - MARK_PRICE
  - LAST_PRICE
    
**Payload:**

```javascript
{
    "e": "ORDER_TRADE_UPDATE",                   // Event Type
    "E": 1719213612441,                          // Event Time
    "T": 1719213612249,                          // Transaction Time
    "o": {
        "s": "BTCUSDT",                          // Symbol
        "c": "c0c5e30e284348679fa9732899628d63", // Client Order Id
        "S": "BUY",                              // Side
        "o": "MARKET",                           // Order Type
        "f": "GTC",                              // Time in Force
        "q": "0.000100000000000000",             // Original Quantity
        "p": "0.000000000000000000",             // Original Price
        "sp": "0.000000000000000000",            // Stop Price. Please ignore with TRAILING_STOP_MARKET order
        "x": "TRADE",                            // Execution Type
        "X": "FILLED",                           // Order Status
        "i": 1715282533938061568,                // Order Id
        "l": "0.000100000000000000",             // Order Last Filled Quantity
        "L": "69000.000000000000000000",         // Last Filled Price
        "z": "0.000100000000000000",             // Order Filled Accumulated Quantity
        "N": "USDT",                             // Commission Asset, will not push if no commission
        "n": "-0.003450000000000000",            // Commission, will not push if no commission
        "T": 1719213612249,                      // Order Trade Time
        "t": 1715282533938061568,                // Trade Id
        "m": false,                              // Is this trade the maker side?
        "r": ""                                  // Order reject reason
        "b": "USDT"                              // Base token
    }
}
```
