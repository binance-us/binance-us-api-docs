# CHANGELOG for Binance US API (2023-09-06)


## 2023-09-06

- Interval for `1s` klines has been removed from the documentation as it is no longer supported.

---

## 2023-05-12

* Spot WebSocket API is now available for Binance US.
    * WebSocket API allows placing orders, canceling orders, etc. through a WebSocket connection.
    * WebSocket API is a **separate** service from WebSocket Market Data streams. I.e., placing orders and listening to market data requires two separate WebSocket connections.
    * WebSocket API is subject to the same Filter and Rate Limit rules as REST API.
    * WebSocket API and REST API are functionally equivalent: they provide the same features, accept the same parameters, return the same status and error codes.
* The full documentation can be found [here](./web-socket-api.md).

---

## 2023-03-08 

**Notice:** All changes are being rolled out gradually to all our servers, and may take a few days to complete.

GENERAL CHANGES

* The error messages for certain issues have been improved for easier troubleshooting.

<table>
    <tr>
        <th>Situation</th>
        <th>Old Error Message</th>
        <th>New Error Message</th>
    </tr>
    <tr>
        <td>An account cannot place or cancel an order, due to trading ability disabled.</td>
        <td rowspan="3">This action is disabled on this account.</td>
        <td>This account may not place or cancel orders.</td>
    </tr>
    <tr>
        <td>When the permissions configured on the symbol do not match the permissions on the account.</td>
        <td>This symbol is not permitted for this account.</td>
    </tr>
    <tr>
        <td>When the account tries to place an order on a symbol it has no permissions for.</td>
        <td>This symbol is restricted for this account.</td>
    </tr>
    <tr>
        <td>Placing an order when <tt>symbol</tt> is not <tt>TRADING</tt>. </td>
        <td rowspan="2">Unsupported order combination.</td>
        <td>This order type is not possible in this trading phase.</td>
    </tr>
    <tr>
        <td>Placing an order with <tt>timeinForce</tt>=<tt>IOC</tt> or <tt>FOK</tt> on a trading phase that does not support it.</td>
        <td>Limit orders require GTC for this phase.</td>
    </tr>
</table>

* Fixed error message for querying archived orders: 
    * Previously, if an archived order (i.e. order with status `CANCELED` or `EXPIRED` where `executedQty` == 0 that occurred in the last 90 days) is queried, the error message would be:
    ```json
    {
        "code": -2013,
        "msg": "Order does not exist." 
    }
    ```
    * Now, the error message is: 
    ```json
    {
        "code": -2026,
        "msg": "Order was canceled or expired with no executed qty over 90 days ago and has been archived." 
    }
    ```
* Behavior for API requests with `startTime` and `endTime`:
    * Previously some requests failed if the `startTime` == `endTime`.
    * Now, all API requests that accept `startTime` and `endTime` allow the parameters to be equal. This applies to the following requests:
        * `GET /api/v3/aggTrades`
        * `GET /api/v3/klines`
        * `GET /api/v3/allOrderList`
        * `GET /api/v3/allOrders`
        * `GET /api/v3/myTrades`

**Note:** The following changes will take effect **approximately one to two days from the release date**, but the rest of the documentation has been updated to reflect the future changes: 

* Changes to Filter Evaluation: 
    * Previous behavior: `LOT_SIZE` and `MARKET_LOT_SIZE` required that (`quantity` - `minQty`) % `stepSize` == 0. 
    * New behavior: This has now been changed to (`quantity` % `stepSize`) == 0.
* Bug fix with reverse `MARKET` orders (i.e. `MARKET` using `quoteOrderQty`):
    * Previous behavior: Reverse market orders would have the status `FILLED` even if the order was not fully filled. 
    * New behavior: If the reverse market order did not fully fill due to low liquidity the order status will be `EXPIRED`, and `FILLED` only if the order was completely filled.

REST API

* Changes to `DELETE /api/v3/order` and `POST /api/v3/order/cancelReplace`:
    * New optional parameter `cancelRestrictions` that determine whether the cancel will succeed if the order status is `NEW` or `PARTIALLY_FILLED`.
    * If the order cancellation fails due to `cancelRestrictions`, the error will be: 
    ```json
    {
        "code": -2011, 
        "msg": "Order was not canceled due to cancel restrictions."
    }
    ```


## 2023-01-24

The changes to the system will take place on **January 31, 2023**.

Additional details on the functionality of STP is explained in the [STP FAQ](./faqs/stp_faq.md) document.

Rest API

* Self Trade Prevention (aka STP) will be added to the system. This will prevent orders from matching with orders from the same account, or accounts under the same `tradeGroupId`. The default and allowed modes can be confirmed using `GET /api/v3/exchangeInfo`: 
```javascript
"defaultSelfTradePreventionMode": "EXPIRE_MAKER",   //If selfTradePreventionMode not provided, this will be the value passed to the engine
"allowedSelfTradePreventionModes": [        //What the allowed modes of selfTradePrevention are
    "EXPIRE_MAKER",
    "EXPIRE_TAKER",
    "EXPIRE_BOTH"
]
```
* New order status: `EXPIRED_IN_MATCH` - This means that the order expired due to STP being triggered.
* New endpoints:
   * `GET /api/v3/myPreventedMatches` - This queries the orders that expired due to STP being triggered.
* New optional parameter `selfTradePreventionMode` has been added to the following endpoints:
    * `POST /api/v3/order`
    * `POST /api/v3/order/oco`
    * `POST /api/v3/cancelReplace`
* New responses that will appear for all order placement endpoints if there was a prevented match (i.e. if an order could have matched with an order of the same account, or the accounts are in the same `tradeGroupId`): 
    * `tradeGroupId`      - This will only appear if account is configured to a `tradeGroupId` and if there was a prevented match.
    * `preventedQuantity` - Only appears if there was a prevented match
    * An array `preventedMatches` with the following fields:
        * `preventedMatchId`
        * `makerOrderId`
        * `price`
        * `takerPreventedQuantity` - This will only appear if `selfTradePreventionMode` set is `EXPIRE_TAKER` or `EXPIRE_BOTH`.
        * `makerPreventedQuantity` - This will only appear if `selfTradePreventionMode` set is `EXPIRE_MAKER` or `EXPIRE_BOTH`.
* New fields `preventedMatchId` and `preventedQuantity` that can appear in the order query endpoints if the order had expired due to STP trigger: 
    * `GET /api/v3/order`
    * `GET /api/v3/openOrders`
    * `GET /api/v3/allOrders`
* New field `tradeGroupId` will appear in the `GET /api/v3/account` response.

USER DATA STREAM

* New execution Type: `TRADE_PREVENTION`
* New fields for `executionReport` (These fields will only appear if the order has expired due to STP trigger)
    * `u` - `tradeGroupId` 
    * `v` - `preventedMatchId`
    * `U` - `counterOrderId`
    * `A` - `preventedQuantity`
    * `B` - `lastPreventedQuantity`

---

## 2022-11-28

**Notice:** These changes are being rolled out gradually to all our servers, and will take approximately a week to complete.

WEBSOCKET

*  `!bookTicker` has been removed. Please use the Individual Book Ticker Streams instead. (`<symbol>@bookTicker`).
    * Multiple `<symbol>@bookTicker` streams can be subscribed to over one connection. (E.g. `wss://stream.binance.us:9443/stream?streams=btcusdt@bookTicker/bnbbtc@bookTicker`)
    * This removal will take effect **TOMORROW (2022-11-29)**.

REST API 

* New error code `-1135`
    * This error code will occur if a parameter requiring a JSON object is invalid.
* New error code `-1108`
    * This error will occur if a value to a parameter being sent was too large, potentially causing overflow.
    * This error code can occur in the following endpoints:
        * `POST /api/v3/order`
        * `POST /api/v3/order/cancelReplace`
        * `POST /api/v3/order/oco`
* Changes to `GET /api/v3/aggTrades`
    * Previous behavior: `startTime` and `endTime` had to be used in combination and could only be an hour apart.
    * New behavior: `startTime` and `endTime` can be used individually and the 1 hour limit has been removed.
        * When using `startTime` only, this will return trades from that time, up to the `limit` provided.
        * When using `endTime` only, this will return trades starting from the `endTime` including all trades before that time, up to the limit provided.
        * If `limit` not provided, regardless of used in combination or sent individually, the endpoint will use the default limit.
* Changes to `GET /api/v3/myTrades`
    * Fixed a Bug where combining `symbol` + `orderId` combination would returns all trades even if the number of trades went beyond the `500` default limit. Now if this combination is used the trade will not go beyond the default `limit`.
    * New behavior when sending combination of optional parameters that were not supported:
        * Previous behavior: The API would send specific error messages depending on the combination of parameters sent. Eg.
        ```json
        { "code": -1106, "msg": "Parameter X was sent when not required." } 
        ```
        * New behavior: If the combinations of optional parameters to the endpoint were not supported, then the endpoint will respond with the generic error:
        ```json
        { "code": -1128, "msg": "Combination of optional parameters invalid." }
        ```
    * Added a new combination of supported parameters: `symbol` + `orderId` + `fromId`.
    * The following combinations of parameters were previously supported but now **removed**.
        * `symbol` + `fromId` + `startTime`
        * `symbol` + `fromId` + `endTime`
        * `symbol` + `fromId` + `startTime` + `endTime`
    * Thus, these are the supported combinations of parameters:
        * `symbol`
        * `symbol` + `orderId`
        * `symbol` + `startTime`
        * `symbol` + `endTime`
        * `symbol` + `fromId`
        * `symbol` + `startTime` + `endTime`
        * `symbol`+ `orderId` + `fromId`

**Note:** These new fields will appear approximately a week from the release date.

* Changes to `GET /api/v3/exchangeInfo`
    * New fields `defaultSelfTradePreventionMode` and `allowedSelfTradePreventionModes`
* Changes to the Order Placement Endpoints/Order Query/Order Cancellation Endpoints:
    * New field `selfTradePreventionMode` will appear in the response.
    * Affects the following endpoints:
        * `POST /api/v3/order`
        * `POST /api/v3/order/oco`
        * `POST /api/v3/order/cancelReplace`
        * `GET /api/v3/order`
        * `DELETE /api/v3/order`
        * `DELETE /api/v3/orderList`
* Changes to  `GET /api/v3/account`
    * New field `requireSelfTradePrevention` will appear in the response.
* New field `workingTime`, indicating when the order started working on the order book, will appear in the following endpoints:
    * `POST /api/v3/order`
    * `GET /api/v3/order`
    * `POST /api/v3/order/cancelReplace`
    * `POST /api/v3/order/oco`
    * `GET /api/v3/order`
    * `GET /api/v3/openOrders`
    * `GET /api/v3/allOrders`
* Field `trailingTime`, indicating the time when the trailing order is active and tracking price changes, will appear for the following order types  (`TAKE_PROFIT`, `TAKE_PROFIT_LIMIT`, `STOP_LOSS`, `STOP_LOSS_LIMIT` if `trailingDelta` parameter was provided) for the following endpoints: 
    * `POST /api/v3/order`
    * `GET /api/v3/order`
    * `GET /api/v3/openOrders`
    * `GET /api/v3/allOrders`
    * `POST /api/v3/order/cancelReplace`
    * `DELETE /api/v3/order`
* Field `commissionRates` will appear in the `GET /api/v3/acccount` response

USER DATA STREAM

* eventType `executionReport` has new fields
    * `V` - `selfTradePreventionMode` 
    * `D` - `trailing_time`  (Appears if the trailing stop order is active)
    * `W` - `workingTime`   (Appears if the order is working on the order book)

---

## 2022-10-11

* Trailing Stops have been enabled.
    * This is a type of algo order where the activation is based on a percentage of a price change in the market using the new parameter `trailingDelta`.
    * This can only be used with any of the following order types: `STOP_LOSS`, `STOP_LOSS_LIMIT`, `TAKE_PROFIT`, `TAKE_PROFIT_LIMIT`.
    * The `trailingDelta` parameter will be done in Basis Points or BIPS.
        * For example: a STOP_LOSS SELL order with a `trailingDelta` of 100 will trigger after a price decrease of 1% from the highest price after the order is placed. (100 / 10,000 => 0.01 => 1%)
    * When used in combination with OCO Orders, the `trailingDelta` will determine when the contingent leg of the OCO will trigger.
    * When `trailingDelta` is used in combination with `stopPrice`, once the `stopPrice` condition is met, the trailing stop starts tracking the price change from the `stopPrice` based on the `trailingDelta` provided.
    * When no `stopPrice` is sent, the trailing stop starts tracking the price changes from the last price based on the `trailingDelta` provided.
* Changes to POST `/api/v3/order`
    * New optional field `trailingDelta`
* Changes to POST `/api/v3/order/test`
    * New optional field `trailingDelta`
* Changes to POST `/api/v3/order/oco`
    * New optional field `trailingDelta`
* A new filter `TRAILING_DELTA` has been added.
    * This filter is defined by the minimum and maximum values for the `trailingDelta` value.
* Cancel Replace Orders have been enabled.
    * Cancels an existing order and places a new order on the same symbol using the endpoint `POST /api/v3/order/cancelReplace`

---

## 2022-09-30

Scheduled changes to the removal of `!bookTicker` around November 2022.

* The All Book Tickers stream (`!bookTicker`) is set to be removed in **November 2022**
* More details of the actual removal date will be announced at a later time.
* Please use the Individual Book Ticker Streams instead. (`<symbol>@bookTicker`)
* Multiple `<symbol>@bookTicker` streams can be subscribed to over one connection. 
    * Example: wss://stream.binance.us:9443/stream?streams=btcusdt@bookTicker/bnbbtc@bookTicker

---

## 2022-09-14

Note that these are rolling changes, so it may take a few days for it to rollout to all our servers.

* Changes to `GET /api/v3/exchangeInfo`
    * New optional parameter `permissions` added to display all symbols with the permissions matching the value provided. 
    * If not provided, the default value will be `["SPOT"]`
    * Cannot be combined with `symbol` or `symbols`

---

## 2022-08-22

* Changes to `GET /api/v3/ticker` and `GET /api/v3/ticker/24hr`
    * New optional parameter `type` added
    * Supported values for parameter `type` are `FULL` and `MINI`
        * `FULL` is the default value and the response that is currently being returned from the endpoint
        * `MINI` omits the following fields from the response: `priceChangePercent`, `weightedAvgPrice`, `bidPrice`, `bidQty`, `askPrice`, `askQty`, and `lastQty`
* New error code `-1008` 
    * This is sent whenever the servers are overloaded with requests.
* New field `brokered` has been added to `GET /api/v3/account`
* New kline interval: `1s`

---

## 2022-06-20

Changes to `GET /api/v3/ticker`

* Weight has been reduced from 5 to 2 per symbol, regardless of `windowSize`. 
* The max number of symbols that can be processed in a request is 100.
    * If the number of `symbols` sent is more than 100, the error will be as follows:
    ```json
    {
     "code": -1101,
     "msg": "Too many values sent for parameter 'symbols', maximum allowed up to 100." 
    }
    ```
* The max Weight for this endpoint will cap at 100.
    * I.e. If the request has more than 50 symbols, the Weight will still be 100, regardless of `windowSize`.

---
## 2022-06-13

Rest API

* `GET /api/v3/ticker` added
    * Rolling window price change statistics based on `windowSize` provided.
    * Contrary to `GET /api/v3/ticker/24hr` the list of symbols cannot be omitted.
    * If `windowSize` not specified, the default value will default to `1d`.
    * Response is similar to `GET /api/v3/ticker/24hr`, minus the following fields: `prevClosePrice`, `lastQty`, `bidPrice`, `bidQty`, `askPrice`, `askQty`
* `GET /api/v3/exchangeInfo` returns new field `cancelReplaceAllowed` in `symbols` list. 
* `POST /api/v3/order/cancelReplace` added
    * Cancels an existing order and places a new order on the same symbol.
    * The filters are evaluated **before** the cancel order is placed.
        * e.g. If the `MAX_NUM_ORDERS` filter is 10, and the total number of open orders on the account is also 10, when using `POST /api/v3/order/cancelReplace` both the cancel order placement and new order will fail because of the filter.
* New filter `NOTIONAL` has been added.
    * Defines the allowed notional value (`price * quantity`) based on a configured `minNotional` and `maxNotional`
* New exchange filter `EXCHANGE_MAX_NUM_ICEBERG_ORDERS` has been added.
    * Defines the limit of open iceberg orders on an account

WEBSOCKETS

* New symbol ticker streams with `1h` and `4h` windows:
    * Individual symbol ticker streams
        * `<symbol>@ticker_<window-size>`
    * All market ticker streams
        * `!ticker_<window-size>@arr`

## 2022-03-31

REST API

* Changes to `GET api/v3/allOrders` where symbol is not provided:
    ```json
    {
     "code": -1102,
     "msg": "Mandatory parameter 'symbol' was not sent, was empty/null, or malformed."
    }
    ```
* Fixed a typo with an error message when an account has disabled permissions (e.g. to withdraw, to trade, etc)
    ```json
    "This action is disabled on this account." 
    ```
    
---

## 2022-02-28

* New field `allowTrailingStop` has been added to GET /api/v3/exchangeInfo

---

## 2022-02-25

**REST API**

* `(price-minPrice) % tickSize == 0` rule in `PRICE_FILTER` has been changed to `price % tickSize == 0`.
* A new filter `PERCENT_PRICE_BY_SIDE` has been added.
* Changes to GET `api/v3/depth`
    * The `limit` value can be outside of the previous values (i.e. 5, 10, 20, 50, 100, 500, 1000,5000) and will return the correct limit. (i.e. if limit=3 then the response will be the top 3 bids and asks)
    * The limit still cannot exceed 5000. If the limit provided is greater than 5000, then the response will be truncated to 5000.
    * Due to the changes, these are the updated request weights based on the limit value provided:

|Limit|Request Weight
------|-------
1-100|  1
101-500| 5
501-1000| 10
1001-5000| 50

* Changes to GET `api/v3/aggTrades`
    * When providing `startTime` and `endTime`, the oldest items are returned.

* Changed error messaging on `GET /api/v3/myTrades` where parameter `symbol` is not provided:
```json
{
"code": -1102,
"msg": "Mandatory parameter 'symbol' was not sent, was empty/null, or malformed." 
}

```
*  The following endpoints now support multi-symbol querying using the parameter `symbols`.
    * `GET /api/v3/ticker/24hr`
    * `GET /api/v3/ticker/price`
    * `GET /api/v3/ticker/bookTicker`
* In the above, the request weight will depend on the number of symbols provided in `symbols`. <br> Please refer to the table below:

|Endpoint|Number of Symbols|Weight|
|-----|-----|----|
| `GET /api/v3/ticker/price`|Any| 2|
|`GET /api/v3/ticker/bookTicker`|Any|2|
|`GET /api/v3/ticker/24hr`|1-20|1|
|`GET /api/v3/ticker/24hr`|21-100|20|
|`GET /api/v3/ticker/24hr`|101 or more|40|

---

## 2022-02-01
* GET `api/v3/rateLimit/order` added
    * The endpoint will display the user's current order count usage for all intervals.
    * This endpoint will have a request weight of 20.
* Removed out dated "Symbol Type" enum; added "Permissions" enum.

---

## 2021-08-12
* GET `api/v3/myTrades` has a new optional field `orderId`

---
## 2021-06-21

### USER DATA STREAM
* `outboundAccountInfo` has been removed

### REST API
* `GET api/v3/exchangeInfo` now supports single and multi-symbol query.
* The weights to the following endpoints will be adjusted:
    * `GET /api/v3/order` weight increased to 2
    * `GET /api/v3/openOrders` weight increased to 3
    * `GET /api/v3/allOrders` weight increased to 10
    * `GET /api/v3/orderList` weight increased to 2
    * `GET /api/v3/openOrderList` weight increased to 3
    * `GET /api/v3/account` weight increased to 10
    * `GET /api/v3/myTrades` weight increased to 10
    * `GET /api/v3/exchangeInfo` weight increased to 10

---
## 2021-05-12
* Added `Data Source` in the documentation to explain where each endpoint is retrieving its data.
* Added field `Data Source` to each API endpoint in the documentation

---
## 2020-07-09

### REST API
* `api/v3/exchangeInfo` has new fields:
    * `quoteOrderQtyMarketAllowed`
        * Indicates whether Quote Order Qty `MARKET` orders are enabled on a symbol.
    * `quoteAssetPrecision`
        * This is a duplicate of the `quotePrecision` field. `quotePrecision` will be removed in future API versions (v4+).
* MARKET orders have a new optional field: `quoteOrderQty` to specify the total `quoteOrderQty` spent or received on a `MARKET` order.
    * Quote Order Qty `MARKET` orders will not break `LOT_SIZE` filter rules; the order will execute a quantity that will have the notional value as close as possible to the `quoteOrderQty`
    * Using `BNBBTC` as an example:
        * On the `BUY` side, the order will buy as many BNB as `quoteOrderQty` BTC can.
        * On the `SELL` side, the order will sell as much BNB as needed to receive `quoteOrderQty` BTC.
* All order query endpoints will return a new field `origQuoteOrderQty` in the JSON payload. (e.g. GET api/v3/allOrders)
* New field `permissions`
    * Defines the trading permissions that allowed on accounts and symbols
    * `permissions` is an enum array;values:
        * `SPOT`
        * `MARGIN`
    * `permissions` will replace `isSpotTradingAllowed` and `isMarginTradingAllowed` on `GET api/v3/exchangeInfo` in future API versions (v4+).
    * For an account to trade on a symbol, the account and symbol must share at least 1 permission in common.
* Updates to `GET api/v3/account`
    * New field `permissions` added.
* New endpoint `DELETE /api/v3/openOrders`
    * This will allow a user to cancel all open orders on a single symbol.
    * This will cancel all open orders including OCO orders.
* Updated error messages for -1003 to specify the limit is referring to the request weight, not to the number of requests.
* Updated error messages for -1128:
    * Sending an `OCO` with a `stopLimitPrice` without a `stopLimitTimeInForce` will return this message:

   ```javascript
      {
        "code": -1128,
        "msg": "Combination of optional parameters invalid. Recommendation: 'stopLimitTimeInForce' should also be sent."
      }
   ```
* Orders can be canceled via the API on symbols in the `BREAK` or `HALT` status.

### USER DATA STREAM

* `OutboundAccountInfo` has a new field `P` which shows the trading permissions of the account.
* `executionReport` has a new field `Q` which shows the quoteOrderQty of an order.
* `balanceUpdate` event type added
    * This event occurs when funds are deposited or withdrawn from the account. 

---
## 2020-04-23
### WEB SOCKET STREAM

* WebSocket connections have a limit of 5 incoming messages per second. A message is considered:
    * A PING frame
    * A PONG frame
    * A JSON control message (e.g. subscribe, unsubscribe)
* A connection that goes beyond the limit will be disconnected; IPs that are repeatedly disconnected may be banned.
* A single connection can listen to a maximum of 1024 streams.

---
## 2019-11-13
### WEB SOCKET STREAM
* WSS now supports live subscribing/unsubscribing to streams.
---
## 2019-09-09
* New WebSocket streams for bookTickers added: `<symbol>@bookTicker` and `!bookTicker`. See `web-socket-streams.md` for details.

---
## 2019-09-03
* Faster order book data with 100ms updates: `<symbol>@depth@100ms` and `<symbol>@depth#@100ms`
* Added "Update Speed:" to `web-socket-streams.md`
* Removed deprecated v1 endpoints as per previous announcement:
    * GET api/v1/order
    * GET api/v1/openOrders
    * POST api/v1/order
    * DELETE api/v1/order
    * GET api/v1/allOrders
    * GET api/v1/account
    * GET api/v1/myTrades

---
## 2019-08-16 (Update 2)
* GET api/v1/depth `limit` of 10000 has been temporarily removed

---
## 2019-08-16
* In Q4 2017, the following endpoints were deprecated and removed from the API documentation. They have been permanently removed from the API as of this version. We apologize for the omission from the original changelog:
    * GET api/v1/order
    * GET api/v1/openOrders
    * POST api/v1/order
    * DELETE api/v1/order
    * GET api/v1/allOrders
    * GET api/v1/account
    * GET api/v1/myTrades

* Streams, endpoints, parameters, payloads, etc. described in the documents in this repository are **considered official** and **supported**. The use of any other streams, endpoints, parameters, or payloads, etc. is **not supported; use them at your own risk and with no guarantees.**

---
## 2019-08-15
### Rest API
* New order type: OCO ("One Cancels the Other")
    * An OCO has 2 orders: (also known as legs in financial terms)
        * ```STOP_LOSS``` or ```STOP_LOSS_LIMIT``` leg
        * ```LIMIT_MAKER``` leg

    * Price Restrictions:
        * ```SELL Orders``` : Limit Price > Last Price > Stop Price
        * ```BUY Orders``` : Limit Price < Last Price < Stop Price
        * As stated, the prices must "straddle" the last traded price on the symbol. EX: If the last price is 10:
            * A SELL OCO must have the limit price greater than 10, and the stop price less than 10.
            * A BUY OCO must have a limit price less than 10, and the stop price greater than 10.

    * Quantity Restrictions:
        * Both legs must have the **same quantity**.
        * ```ICEBERG``` quantities however, do not have to be the same.

    * Execution Order:
        * If the ```LIMIT_MAKER``` is touched, the limit maker leg will be executed first BEFORE canceling the Stop Loss Leg.
        * if the Market Price moves such that the ```STOP_LOSS``` or ```STOP_LOSS_LIMIT``` will trigger, the Limit Maker leg will be canceled BEFORE executing the ```STOP_LOSS``` Leg.

    * Canceling an OCO
        * Canceling either order leg will cancel the entire OCO.
        * The entire OCO can be canceled via the ```orderListId``` or the ```listClientOrderId```.

    * New Enums for OCO:
        1. ```ListStatusType```
            * ```RESPONSE``` - used when ListStatus is responding to a failed action. (either order list placement or cancellation)
            * ```EXEC_STARTED``` - used when an order list has been placed or there is an update to a list's status.
            * ```ALL_DONE``` - used when an order list has finished executing and is no longer active.
        1. ```ListOrderStatus```
            * ```EXECUTING``` - used when an order list has been placed or there is an update to a list's status.
            * ```ALL_DONE``` - used when an order list has finished executing and is no longer active.
            * ```REJECT``` - used when ListStatus is responding to a failed action. (either order list placement or cancellation)
        1. ```ContingencyType```
            * ```OCO``` - specifies the type of order list.

    * New Endpoints:
        * POST api/v3/order/oco
        * DELETE api/v3/orderList
        * GET api/v3/orderList

* ```recvWindow``` cannot exceed 60000.
* New `intervalLetter` values for headers:
    * SECOND => S
    * MINUTE => M
    * HOUR => H
    * DAY => D
* New Headers `X-MBX-USED-WEIGHT-(intervalNum)(intervalLetter)` will give your current used request weight for the (intervalNum)(intervalLetter) rate limiter. For example, if there is a one minute request rate weight limiter set, you will get a `X-MBX-USED-WEIGHT-1M` header in the response. The legacy header `X-MBX-USED-WEIGHT` will still be returned and will represent the current used weight for the one minute request rate weight limit.
* New Header `X-MBX-ORDER-COUNT-(intervalNum)(intervalLetter)`that is updated on any valid order placement and tracks your current order count for the interval; rejected/unsuccessful orders are not guaranteed to have `X-MBX-ORDER-COUNT-**` headers in the response.
    * Eg. `X-MBX-ORDER-COUNT-1S` for "orders per 1 second" and `X-MBX-ORDER-COUNT-1D` for orders per "one day"
* GET api/v1/depth now supports `limit` 5000 and 10000; weights are 50 and 100 respectively.
* GET api/v1/exchangeInfo has a new parameter `ocoAllowed`.

### USER DATA STREAM
* ```executionReport``` event now contains "g" which has the ```orderListId```; it will be set to -1 for non-OCO orders.
* New Event Type ```listStatus```; ```listStatus``` is sent on an update to any OCO order.
* New Event Type ```outboundAccountPosition```; ```outboundAccountPosition``` is sent any time an account's balance changes and contains the assets that could have changed by the event that generated the balance change (a deposit, withdrawal, trade, order placement, or cancellation).

### NEW ERRORS
* **-1131 BAD_RECV_WINDOW**
    * ```recvWindow``` must be less than 60000
* **-1099 Not found, authenticated, or authorized**
    * This replaces error code -1999

### NEW -2011 ERRORS
* **OCO_BAD_ORDER_PARAMS**
    * A parameter for one of the orders is incorrect.
* **OCO_BAD_PRICES**
    * The relationship of the prices for the orders is not correct.
* **UNSUPPORTED_ORD_OCO**
    * OCO orders are not supported for this symbol.

---
## 2019-03-12
### Rest API
* X-MBX-USED-WEIGHT header added to Rest API responses.
* Retry-After header added to Rest API 418 and 429 responses.
* When canceling the Rest API can now return `errorCode` -1013 OR -2011 if the symbol's `status` isn't `TRADING`.
* `api/v1/depth` no longer has the ignored and empty `[]`.
* `api/v3/myTrades` now returns `quoteQty`; the price * qty of for the trade.
  
### Websocket streams
* `<symbol>@depth` and `<symbol>@depthX` streams no longer have the ignored and empty `[]`.
  
### System improvements
* Matching Engine stability/reliability improvements.
* Rest API performance improvements.

---
## 2018-11-13
### Rest API
* Can now cancel orders through the Rest API during a trading ban.
* New filters: `PERCENT_PRICE`, `MARKET_LOT_SIZE`, `MAX_NUM_ICEBERG_ORDERS`.
* Added `RAW_REQUST` rate limit. Limits based on the number of requests over X minutes regardless of weight.
* /api/v3/ticker/price increased to weight of 2 for a no symbol query.
* /api/v3/ticker/bookTicker increased weight of 2 for a no symbol query.
* DELETE /api/v3/order will now return an execution report of the final state of the order.
* `MIN_NOTIONAL` filter has two new parameters: `applyToMarket` (whether or not the filter is applied to MARKET orders) and `avgPriceMins` (the number of minutes over which the price averaged for the notional estimation).
* `intervalNum` added to /api/v1/exchangeInfo limits. `intervalNum` describes the amount of the interval. For example: `intervalNum` 5, with `interval` minute, means "every 5 minutes".
  
#### Explanation for the average price calculation:
1. (qty * price) of all trades / sum of qty of all of the trades over previous 5 minutes.

2. If there is no trade in the last 5 minutes, it takes the first trade that happened outside of the 5min window.
   For example if the last trade was 20 minutes ago, that trade's price is the 5 min average.

3. If there is no trade on the symbol, there is no average price and market orders cannot be placed.
   On a new symbol with `applyToMarket` enabled on the `MIN_NOTIONAL` filter, market orders cannot be placed until there is at least 1 trade.

4. The current average price can be checked here: `https://api.binance.us/api/v3/avgPrice?symbol=<symbol>`
   For example:
   https://api.binance.us/api/v3/avgPrice?symbol=BNBUSDT

### User data stream
* `Last quote asset transacted quantity` (as variable `Y`) added to execution reports. Represents the `lastPrice` * `lastQty` (`L` * `l`).

---
## 2018-07-18
### Rest API
*  New filter: `ICEBERG_PARTS`
*  `POST api/v3/order` new defaults for `newOrderRespType`. `ACK`, `RESULT`, or `FULL`; `MARKET` and `LIMIT` order types default to `FULL`, all other orders default to `ACK`.
*  POST api/v3/order `RESULT` and `FULL` responses now have `cummulativeQuoteQty`
*  GET api/v3/openOrders with no symbol weight reduced to 40.
*  GET api/v3/ticker/24hr with no symbol weight reduced to 40.
*  Max amount of trades from GET /api/v1/trades increased to 1000.
*  Max amount of trades from GET /api/v1/historicalTrades increased to 1000.
*  Max amount of aggregate trades from GET /api/v1/aggTrades increased to 1000.
*  Max amount of aggregate trades from GET /api/v1/klines increased to 1000.
*  Rest API Order lookups now return `updateTime` which represents the last time the order was updated; `time` is the order creation time.
*  Order lookup endpoints will now return `cummulativeQuoteQty`. If `cummulativeQuoteQty` is < 0, it means the data isn't available for this order at this time.
*  `REQUESTS` rate limit type changed to `REQUEST_WEIGHT`. This limit was always logically request weight and the previous name for it caused confusion.

### User data stream
*  `cummulativeQuoteQty` field added to order responses and execution reports (as variable `Z`). Represents the cummulative amount of the `quote` that has been spent (with a `BUY` order) or received (with a `SELL` order). Historical orders will have a value < 0 in this field indicating the data is not available at this time. `cummulativeQuoteQty` divided by `cummulativeQty` will give the average price for an order.
*  `O` (order creation time) added to execution reports

---
## 2018-01-23
* GET /api/v1/historicalTrades weight decreased to 5
* GET /api/v1/aggTrades weight decreased to 1
* GET /api/v1/klines weight decreased to 1
* GET /api/v1/ticker/24hr all symbols weight decreased to number of trading symbols / 2
* GET /api/v3/allOrders weight decreased to 5
* GET /api/v3/myTrades weight decreased to 5
* GET /api/v3/account weight decreased to 5
* GET /api/v1/depth limit=500 weight decreased to 5
* GET /api/v1/depth limit=1000 weight decreased to 10
* -1003 error message updated to direct users to websockets

---
## 2018-01-20
* GET /api/v1/ticker/24hr single symbol weight decreased to 1
* GET /api/v3/openOrders all symbols weight decreased to number of trading symbols / 2
* GET /api/v3/allOrders weight decreased to 15
* GET /api/v3/myTrades weight decreased to 15
* GET /api/v3/order weight decreased to 1
* myTrades will now return both sides of a self-trade/wash-trade

---
## 2018-01-14
* GET /api/v1/aggTrades weight changed to 2
* GET /api/v1/klines weight changed to 2
* GET /api/v3/order weight changed to 2
* GET /api/v3/allOrders weight changed to 20
* GET /api/v3/account weight changed to 20
* GET /api/v3/myTrades weight changed to 20
* GET /api/v3/historicalTrades weight changed to 20
