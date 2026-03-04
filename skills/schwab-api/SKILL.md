---
name: schwab-api
description: Use this skill when building applications using Charles Schwab Trading API.
---

# Schwab API Guide

## Table of Contents
1. [Authentication & Security](#authentication--security)
2. [General API Rules & Quirks](#general-api-rules--quirks)
3. [Accounts, Orders & Transactions](#accounts-orders--transactions)
4. [Market Data API](#market-data-api)
5. [Streaming Data (WebSocket)](#streaming-data-websocket)
6. [Common HTTP Error Responses](#common-http-error-responses)
7. [Schemas](#schemas)

## Authentication & Security (OAuth 2.0)

Schwab uses OAuth 2.0 to provide secure, delegated access to user data without exposing credentials. This guide follows the standard Three-Legged OAuth workflow required for the Trader API.

### **1. Application Registration**

Register your application on the [Schwab Developer Portal](https://developer.schwab.com/).

*   **App Status:** New or modified apps enter an `Approved - Pending` state. You cannot use the API until this status changes to `Ready for Use` (typically takes several business days).
*   **Callback URL:** Must be `HTTPS`.
    *   **Host:** Use `127.0.0.1` instead of `localhost` for local development to avoid DNS and IPv6 resolution issues.
    *   **Port:** Use a non-standard port number higher than `1024` (e.g., `https://127.0.0.1:8182`). 
    *   **Important:** Avoid the default HTTPS port (`443`) or port `80`. Most operating systems require superuser/root privileges to listen on ports below `1024`, and firewalls are more likely to block them. Ensure the port number is included in your registration on the Schwab Developer Portal.
*   **Credentials:** You will receive a **Client ID** (App Key) and **Client Secret**. Keep these secure.

### **2. The OAuth Flow**

#### **Step A: Generate Authorization URL**
Construct the URL to redirect the user to Schwab's Login Micro Site (LMS).

**URL:** `https://api.schwabapi.com/v1/oauth/authorize`
**Query Parameters:**
* `client_id`: Your App Key.
* `redirect_uri`: Must exactly match your registered callback URL.

```python
import urllib.parse

app_key = "YOUR_APP_KEY"
callback_url = "https://127.0.0.1:8182"

params = {
    "client_id": app_key,
    "redirect_uri": callback_url
}
auth_url = f"https://api.schwabapi.com/v1/oauth/authorize?{urllib.parse.urlencode(params)}"
print(f"Open this URL in your browser: {auth_url}")
```

#### **Step B: Handle Callback & Extract Code**
After the user approves access, they are redirected to your `redirect_uri` with a `code` parameter. 
*Example:* `https://127.0.0.1:8182/?code=DEF...&session=XYZ...`

**Important:** The `code` must be **URL decoded** (e.g., changing `%40` back to `@`) before use in the next step.

#### **Step C: Exchange Code for Tokens**
Exchange the authorization code for an `access_token` and `refresh_token`.

**Endpoint:** `POST https://api.schwabapi.com/v1/oauth/token`
**Authentication:** Requires `Basic` auth header: `Base64(AppKey:AppSecret)`.

```python
import httpx
import base64

def get_tokens(app_key, app_secret, code, callback_url):
    # Construct Basic Auth Header
    auth_str = f"{app_key}:{app_secret}"
    auth_header = base64.b64encode(auth_str.encode()).decode()
    
    headers = {
        "Authorization": f"Basic {auth_header}",
        "Content-Type": "application/x-www-form-urlencoded"
    }
    
    data = {
        "grant_type": "authorization_code",
        "code": code,
        "redirect_uri": callback_url
    }
    
    response = httpx.post("https://api.schwabapi.com/v1/oauth/token", headers=headers, data=data)
    return response.json()
```

### **3. Token Management & Refresh**

#### **Token Lifecycles**
*   **Access Token:** Valid for **30 minutes**.
*   **Refresh Token:** Valid for **7 days**. 

#### **Proactive Refresh Best Practices**
*   **Access Token:** Refresh when within **60 seconds** of expiration to ensure seamless execution of long-running tasks.
*   **Refresh Token:** Schwab will invalidate the client if the 7-day window expires. Proactively refresh or re-authenticate at **6.5 days**.
*   **Concurrency Control:** If multiple processes share a token, use atomic locking (e.g., SQLite `BEGIN EXCLUSIVE`) during refresh. If two processes attempt to refresh simultaneously, Schwab may invalidate the refresh token.

#### **How to Refresh**
```python
def refresh_access_token(app_key, app_secret, refresh_token):
    auth_str = f"{app_key}:{app_secret}"
    auth_header = base64.b64encode(auth_str.encode()).decode()
    
    headers = {
        "Authorization": f"Basic {auth_header}",
        "Content-Type": "application/x-www-form-urlencoded"
    }
    
    data = {
        "grant_type": "refresh_token",
        "refresh_token": refresh_token
    }
    
    response = httpx.post("https://api.schwabapi.com/v1/oauth/token", headers=headers, data=data)
    return response.json()
```

### **4. Troubleshooting Common Errors**

*   **`401 Unauthorized` during login:** Usually indicates the app is still in `Approved - Pending` status.
*   **`invalid_client`:** The refresh token has expired (over 7 days old) or was invalidated by a subsequent refresh. You must restart the flow from Step A.
*   **SSL Warnings:** When using a local server to capture the callback (like `schwab-py` does), you may see "self-signed certificate" warnings. This is expected for `https://127.0.0.1` as CA-signed certs cannot be issued for localhost. Verify the domain in the address bar before proceeding.

## General API Rules & Quirks

Before placing orders or utilizing the API heavily, you should be aware of several strict limitations and data formatting quirks enforced by Schwab:

* **Account Hashes vs. Account Numbers:** Almost all endpoints related to account balances, positions, and order placement require an **Account Hash** rather than your actual account number. You must first call the `GET /accounts/accountNumbers` endpoint to securely map your raw account number to its hashed counterpart, and pass this hash in the URI paths.
* **Strict Rate Limits:** The API enforces hard rate limits:
 * **120 API requests per minute.**
 * **4000 order-related API calls per day.**
 * **Maximum of 500 concurrently streamed tickers.**
 Exceeding these limits will result in `HTTP 429 (Too Many Requests)` errors.
* **Fractional Shares:** Fractional share orders are strictly not supported by the API.

## Accounts, Orders & Transactions

### Accounts

#### **GET** `/accounts/accountNumbers`

Get list of account numbers and their encrypted values

Account numbers in plain text cannot be used outside of headers or request/response bodies. As the first step consumers must invoke this service to retrieve the list of plain text/encrypted value pairs, and use encrypted account values for all subsequent calls for any accountNumber request.

#### Parameters

No parameters

#### Responses

**Success Response (200)**: List of valid "accounts", matching the provided input parameters.

**Schema on success**:
```json
[
    {
        "accountNumber": "string",
        "hashValue": "string"
    }
]
```

**Headers**:
- `Schwab-Client-CorrelId`: Correlation Id. Auto generated (string)


#### **GET** `/accounts`

Get linked account(s) balances and positions for the logged in user.

All the linked account information for the user logged in. The balances on these accounts are displayed by default however the positions on these accounts will be displayed based on the "positions" flag.

##### Parameters

No parameters

##### Responses

**Success Response (200)**: List of valid "accounts", matching the provided input parameters.

**Schema on success**:
```json
[
    {
        "securitiesAccount": {
            "accountNumber": "string",
            "roundTrips": 0,
            "isDayTrader": false,
            "isClosingOnlyRestricted": false,
            "pfcbFlag": false,
            "positions": [
                {
                    "shortQuantity": 0,
                    "averagePrice": 0,
                    "currentDayProfitLoss": 0,
                    "currentDayProfitLossPercentage": 0,
                    "longQuantity": 0,
                    "settledLongQuantity": 0,
                    "settledShortQuantity": 0,
                    "agedQuantity": 0,
                    "instrument": {
                        "cusip": "string",
                        "symbol": "string",
                        "description": "string",
                        "instrumentId": 0,
                        "netChange": 0,
                        "type": "SWEEP_VEHICLE"
                    },
                    "marketValue": 0,
                    "maintenanceRequirement": 0,
                    "averageLongPrice": 0,
                    "averageShortPrice": 0,
                    "taxLotAverageLongPrice": 0,
                    "taxLotAverageShortPrice": 0,
                    "longOpenProfitLoss": 0,
                    "shortOpenProfitLoss": 0,
                    "previousSessionLongQuantity": 0,
                    "previousSessionShortQuantity": 0,
                    "currentDayCost": 0
                }
            ],
            "initialBalances": {
                "accruedInterest": 0,
                "availableFundsNonMarginableTrade": 0,
                "bondValue": 0,
                "buyingPower": 0,
                "cashBalance": 0,
                "cashAvailableForTrading": 0,
                "cashReceipts": 0,
                "dayTradingBuyingPower": 0,
                "dayTradingBuyingPowerCall": 0,
                "dayTradingEquityCall": 0,
                "equity": 0,
                "equityPercentage": 0,
                "liquidationValue": 0,
                "longMarginValue": 0,
                "longOptionMarketValue": 0,
                "longStockValue": 0,
                "maintenanceCall": 0,
                "maintenanceRequirement": 0,
                "margin": 0,
                "marginEquity": 0,
                "moneyMarketFund": 0,
                "mutualFundValue": 0,
                "regTCall": 0,
                "shortMarginValue": 0,
                "shortOptionMarketValue": 0,
                "shortStockValue": 0,
                "totalCash": 0,
                "isInCall": 0,
                "unsettledCash": 0,
                "pendingDeposits": 0,
                "marginBalance": 0,
                "shortBalance": 0,
                "accountValue": 0
            },
            "currentBalances": {
                "availableFunds": 0,
                "availableFundsNonMarginableTrade": 0,
                "buyingPower": 0,
                "buyingPowerNonMarginableTrade": 0,
                "dayTradingBuyingPower": 0,
                "dayTradingBuyingPowerCall": 0,
                "equity": 0,
                "equityPercentage": 0,
                "longMarginValue": 0,
                "maintenanceCall": 0,
                "maintenanceRequirement": 0,
                "marginBalance": 0,
                "regTCall": 0,
                "shortBalance": 0,
                "shortMarginValue": 0,
                "sma": 0,
                "isInCall": 0,
                "stockBuyingPower": 0,
                "optionBuyingPower": 0
            },
            "projectedBalances": {
                "availableFunds": 0,
                "availableFundsNonMarginableTrade": 0,
                "buyingPower": 0,
                "buyingPowerNonMarginableTrade": 0,
                "dayTradingBuyingPower": 0,
                "dayTradingBuyingPowerCall": 0,
                "equity": 0,
                "equityPercentage": 0,
                "longMarginValue": 0,
                "maintenanceCall": 0,
                "maintenanceRequirement": 0,
                "marginBalance": 0,
                "regTCall": 0,
                "shortBalance": 0,
                "shortMarginValue": 0,
                "sma": 0,
                "isInCall": 0,
                "stockBuyingPower": 0,
                "optionBuyingPower": 0
            }
        }
    }
]
```

**Headers**:
- `Schwab-Client-CorrelId`: Correlation Id. Auto generated (string)


#### **GET** `/accounts/{accountNumber}`

Get a specific account balance and positions for the logged in user.

Specific account information with balances and positions. The balance information on these accounts is displayed by default but Positions will be returned based on the "positions" flag.

##### Parameters

No parameters

##### Responses

**Success Response (200)**: A valid account, matching the provided input parameters

**Schema on success**:
```json
{
    "securitiesAccount": {
        "accountNumber": "string",
        "roundTrips": 0,
        "isDayTrader": false,
        "isClosingOnlyRestricted": false,
        "pfcbFlag": false,
        "positions": [
            {
                "shortQuantity": 0,
                "averagePrice": 0,
                "currentDayProfitLoss": 0,
                "currentDayProfitLossPercentage": 0,
                "longQuantity": 0,
                "settledLongQuantity": 0,
                "settledShortQuantity": 0,
                "agedQuantity": 0,
                "instrument": {
                    "cusip": "string",
                    "symbol": "string",
                    "description": "string",
                    "instrumentId": 0,
                    "netChange": 0,
                    "type": "SWEEP_VEHICLE"
                },
                "marketValue": 0,
                "maintenanceRequirement": 0,
                "averageLongPrice": 0,
                "averageShortPrice": 0,
                "taxLotAverageLongPrice": 0,
                "taxLotAverageShortPrice": 0,
                "longOpenProfitLoss": 0,
                "shortOpenProfitLoss": 0,
                "previousSessionLongQuantity": 0,
                "previousSessionShortQuantity": 0,
                "currentDayCost": 0
            }
        ],
        "initialBalances": {
            "accruedInterest": 0,
            "availableFundsNonMarginableTrade": 0,
            "bondValue": 0,
            "buyingPower": 0,
            "cashBalance": 0,
            "cashAvailableForTrading": 0,
            "cashReceipts": 0,
            "dayTradingBuyingPower": 0,
            "dayTradingBuyingPowerCall": 0,
            "dayTradingEquityCall": 0,
            "equity": 0,
            "equityPercentage": 0,
            "liquidationValue": 0,
            "longMarginValue": 0,
            "longOptionMarketValue": 0,
            "longStockValue": 0,
            "maintenanceCall": 0,
            "maintenanceRequirement": 0,
            "margin": 0,
            "marginEquity": 0,
            "moneyMarketFund": 0,
            "mutualFundValue": 0,
            "regTCall": 0,
            "shortMarginValue": 0,
            "shortOptionMarketValue": 0,
            "shortStockValue": 0,
            "totalCash": 0,
            "isInCall": 0,
            "unsettledCash": 0,
            "pendingDeposits": 0,
            "marginBalance": 0,
            "shortBalance": 0,
            "accountValue": 0
        },
        "currentBalances": {
            "availableFunds": 0,
            "availableFundsNonMarginableTrade": 0,
            "buyingPower": 0,
            "buyingPowerNonMarginableTrade": 0,
            "dayTradingBuyingPower": 0,
            "dayTradingBuyingPowerCall": 0,
            "equity": 0,
            "equityPercentage": 0,
            "longMarginValue": 0,
            "maintenanceCall": 0,
            "maintenanceRequirement": 0,
            "marginBalance": 0,
            "regTCall": 0,
            "shortBalance": 0,
            "shortMarginValue": 0,
            "sma": 0,
            "isInCall": 0,
            "stockBuyingPower": 0,
            "optionBuyingPower": 0
        },
        "projectedBalances": {
            "availableFunds": 0,
            "availableFundsNonMarginableTrade": 0,
            "buyingPower": 0,
            "buyingPowerNonMarginableTrade": 0,
            "dayTradingBuyingPower": 0,
            "dayTradingBuyingPowerCall": 0,
            "equity": 0,
            "equityPercentage": 0,
            "longMarginValue": 0,
            "maintenanceCall": 0,
            "maintenanceRequirement": 0,
            "marginBalance": 0,
            "regTCall": 0,
            "shortBalance": 0,
            "shortMarginValue": 0,
            "sma": 0,
            "isInCall": 0,
            "stockBuyingPower": 0,
            "optionBuyingPower": 0
        }
    }
}
```

**Headers**:
- `Schwab-Client-CorrelId`: Correlation Id. Auto generated (string)



### Orders

#### **GET** `/accounts/{accountNumber}/orders`

Get all orders for a specific account.  
All orders for a specific account. Orders retrieved can be filtered based on input parameters below. Maximum date range is 1 year.

##### Parameters

No parameters

##### Responses

**Success Response (200)**: A List of orders for the account, matching the provided input parameters

**Schema on success**:
```json
[
    {
        "session": "NORMAL",
        "duration": "DAY",
        "orderType": "MARKET",
        "cancelTime": "2025-11-08T22:40:38.846Z",
        "complexOrderStrategyType": "NONE",
        "quantity": 0,
        "filledQuantity": 0,
        "remainingQuantity": 0,
        "requestedDestination": "INET",
        "destinationLinkName": "string",
        "releaseTime": "2025-11-08T22:40:38.846Z",
        "stopPrice": 0,
        "stopPriceLinkBasis": "MANUAL",
        "stopPriceLinkType": "VALUE",
        "stopPriceOffset": 0,
        "stopType": "STANDARD",
        "priceLinkBasis": "MANUAL",
        "priceLinkType": "VALUE",
        "price": 0,
        "taxLotMethod": "FIFO",
        "orderLegCollection": [
            {
                "orderLegType": "EQUITY",
                "legId": 0,
                "instrument": {
                    "cusip": "string",
                    "symbol": "string",
                    "description": "string",
                    "instrumentId": 0,
                    "netChange": 0,
                    "type": "SWEEP_VEHICLE"
                },
                "instruction": "BUY",
                "positionEffect": "OPENING",
                "quantity": 0,
                "quantityType": "ALL_SHARES",
                "divCapGains": "REINVEST",
                "toSymbol": "string"
            }
        ],
        "activationPrice": 0,
        "specialInstruction": "ALL_OR_NONE",
        "orderStrategyType": "SINGLE",
        "orderId": 0,
        "cancelable": false,
        "editable": false,
        "status": "AWAITING_PARENT_ORDER",
        "enteredTime": "2025-11-08T22:40:38.847Z",
        "closeTime": "2025-11-08T22:40:38.847Z",
        "tag": "string",
        "accountNumber": 0,
        "orderActivityCollection": [
            {
                "activityType": "EXECUTION",
                "executionType": "FILL",
                "quantity": 0,
                "orderRemainingQuantity": 0,
                "executionLegs": [
                    {
                        "legId": 0,
                        "price": 0,
                        "quantity": 0,
                        "mismarkedQuantity": 0,
                        "instrumentId": 0,
                        "time": "2025-11-08T22:40:38.847Z"
                    }
                ]
            }
        ],
        "replacingOrderCollection": [
            "string"
        ],
        "childOrderStrategies": [
            "string"
        ],
        "statusDescription": "string"
    }
]
```

**Headers**:
- `Schwab-Client-CorrelId`: Correlation Id. Auto generated (string)


#### **POST** `/accounts/{accountNumber}/orders`

Place an order for a specific account.

##### Parameters

No parameters

#### **Request body**
The new Order Object.

```json
{
    "session": "NORMAL",
    "duration": "DAY",
    "orderType": "MARKET",
    "cancelTime": "2025-11-08T22:40:38.865Z",
    "complexOrderStrategyType": "NONE",
    "quantity": 0,
    "filledQuantity": 0,
    "remainingQuantity": 0,
    "destinationLinkName": "string",
    "releaseTime": "2025-11-08T22:40:38.865Z",
    "stopPrice": 0,
    "stopPriceLinkBasis": "MANUAL",
    "stopPriceLinkType": "VALUE",
    "stopPriceOffset": 0,
    "stopType": "STANDARD",
    "priceLinkBasis": "MANUAL",
    "priceLinkType": "VALUE",
    "price": 0,
    "taxLotMethod": "FIFO",
    "orderLegCollection": [
        {
            "orderLegType": "EQUITY",
            "legId": 0,
            "instrument": {
                "cusip": "string",
                "symbol": "string",
                "description": "string",
                "instrumentId": 0,
                "netChange": 0,
                "type": "SWEEP_VEHICLE"
            },
            "instruction": "BUY",
            "positionEffect": "OPENING",
            "quantity": 0,
            "quantityType": "ALL_SHARES",
            "divCapGains": "REINVEST",
            "toSymbol": "string"
        }
    ],
    "activationPrice": 0,
    "specialInstruction": "ALL_OR_NONE",
    "orderStrategyType": "SINGLE",
    "orderId": 0,
    "cancelable": false,
    "editable": false,
    "status": "AWAITING_PARENT_ORDER",
    "enteredTime": "2025-11-08T22:40:38.865Z",
    "closeTime": "2025-11-08T22:40:38.865Z",
    "accountNumber": 0,
    "orderActivityCollection": [
        {
            "activityType": "EXECUTION",
            "executionType": "FILL",
            "quantity": 0,
            "orderRemainingQuantity": 0,
            "executionLegs": [
                {
                    "legId": 0,
                    "price": 0,
                    "quantity": 0,
                    "mismarkedQuantity": 0,
                    "instrumentId": 0,
                    "time": "2025-11-08T22:40:38.865Z"
                }
            ]
        }
    ],
    "replacingOrderCollection": [
        "string"
    ],
    "childOrderStrategies": [
        "string"
    ],
    "statusDescription": "string"
}
```
##### Responses

**Success Response (201)**: Empty response body if an order was successfully placed/created.

**Headers**:
- `Schwab-Client-CorrelId`: Correlation Id. Auto generated (string)
- `Location`: Link to the newly created order if order was successfully created. (string)


#### **GET** `/accounts/{accountNumber}/orders/{orderId}`

Get a specific order by its ID, for a specific account

##### Parameters

No parameters

##### Responses

**Success Response (200)**: An order object, matching the input parameters

**Schema on success**:
```json
{
    "session": "NORMAL",
    "duration": "DAY",
    "orderType": "MARKET",
    "cancelTime": "2025-11-08T22:40:38.880Z",
    "complexOrderStrategyType": "NONE",
    "quantity": 0,
    "filledQuantity": 0,
    "remainingQuantity": 0,
    "requestedDestination": "INET",
    "destinationLinkName": "string",
    "releaseTime": "2025-11-08T22:40:38.880Z",
    "stopPrice": 0,
    "stopPriceLinkBasis": "MANUAL",
    "stopPriceLinkType": "VALUE",
    "stopPriceOffset": 0,
    "stopType": "STANDARD",
    "priceLinkBasis": "MANUAL",
    "priceLinkType": "VALUE",
    "price": 0,
    "taxLotMethod": "FIFO",
    "orderLegCollection": [
        {
            "orderLegType": "EQUITY",
            "legId": 0,
            "instrument": {
                "cusip": "string",
                "symbol": "string",
                "description": "string",
                "instrumentId": 0,
                "netChange": 0,
                "type": "SWEEP_VEHICLE"
            },
            "instruction": "BUY",
            "positionEffect": "OPENING",
            "quantity": 0,
            "quantityType": "ALL_SHARES",
            "divCapGains": "REINVEST",
            "toSymbol": "string"
        }
    ],
    "activationPrice": 0,
    "specialInstruction": "ALL_OR_NONE",
    "orderStrategyType": "SINGLE",
    "orderId": 0,
    "cancelable": false,
    "editable": false,
    "status": "AWAITING_PARENT_ORDER",
    "enteredTime": "2025-11-08T22:40:38.881Z",
    "closeTime": "2025-11-08T22:40:38.881Z",
    "tag": "string",
    "accountNumber": 0,
    "orderActivityCollection": [
        {
            "activityType": "EXECUTION",
            "executionType": "FILL",
            "quantity": 0,
            "orderRemainingQuantity": 0,
            "executionLegs": [
                {
                    "legId": 0,
                    "price": 0,
                    "quantity": 0,
                    "mismarkedQuantity": 0,
                    "instrumentId": 0,
                    "time": "2025-11-08T22:40:38.881Z"
                }
            ]
        }
    ],
    "replacingOrderCollection": [
        "string"
    ],
    "childOrderStrategies": [
        "string"
    ],
    "statusDescription": "string"
}
```

**Headers**:
- `Schwab-Client-CorrelId`: Correlation Id. Auto generated (string)


#### **DELETE** `/accounts/{accountNumber}/orders/{orderId}`

Cancel a specific order for a specific account

##### Parameters

No parameters

##### Responses

**Success Response (200)**: Empty response body if an order was successfully canceled.

**Headers**:
- `Schwab-Client-CorrelId`: Correlation Id. Auto generated (string)


#### **PUT** `/accounts/{accountNumber}/orders/{orderId}`

Replace an existing order for an account. The existing order will be replaced by the new order. Once replaced, the old order will be canceled and a new order will be created.

##### Parameters

No parameters

##### Request body

**application/json**

The Order Object.

**Example Value:**

```json
{
    "session": "NORMAL",
    "duration": "DAY",
    "orderType": "MARKET",
    "cancelTime": "2025-11-08T22:40:38.909Z",
    "complexOrderStrategyType": "NONE",
    "quantity": 0,
    "filledQuantity": 0,
    "remainingQuantity": 0,
    "destinationLinkName": "string",
    "releaseTime": "2025-11-08T22:40:38.909Z",
    "stopPrice": 0,
    "stopPriceLinkBasis": "MANUAL",
    "stopPriceLinkType": "VALUE",
    "stopPriceOffset": 0,
    "stopType": "STANDARD",
    "priceLinkBasis": "MANUAL",
    "priceLinkType": "VALUE",
    "price": 0,
    "taxLotMethod": "FIFO",
    "orderLegCollection": [
        {
            "orderLegType": "EQUITY",
            "legId": 0,
            "instrument": {
                "cusip": "string",
                "symbol": "string",
                "description": "string",
                "instrumentId": 0,
                "netChange": 0,
                "type": "SWEEP_VEHICLE"
            },
            "instruction": "BUY",
            "positionEffect": "OPENING",
            "quantity": 0,
            "quantityType": "ALL_SHARES",
            "divCapGains": "REINVEST",
            "toSymbol": "string"
        }
    ],
    "activationPrice": 0,
    "specialInstruction": "ALL_OR_NONE",
    "orderStrategyType": "SINGLE",
    "orderId": 0,
    "cancelable": false,
    "editable": false,
    "status": "AWAITING_PARENT_ORDER",
    "enteredTime": "2025-11-08T22:40:38.909Z",
    "closeTime": "2025-11-08T22:40:38.909Z",
    "accountNumber": 0,
    "orderActivityCollection": [
        {
            "activityType": "EXECUTION",
            "executionType": "FILL",
            "quantity": 0,
            "orderRemainingQuantity": 0,
            "executionLegs": [
                {
                    "legId": 0,
                    "price": 0,
                    "quantity": 0,
                    "mismarkedQuantity": 0,
                    "instrumentId": 0,
                    "time": "2025-11-08T22:40:38.909Z"
                }
            ]
        }
    ],
    "replacingOrderCollection": [
        "string"
    ],
    "childOrderStrategies": [
        "string"
    ],
    "statusDescription": "string"
}
```

##### Responses

**Success Response (201)**: Empty response body if an order was successfully replaced/created.

**Headers**:
- `Schwab-Client-CorrelId`: Correlation Id. Auto generated (string)
- `Location`: Link to the newly created order if order was successfully created. (string)


#### **GET** `/orders`

Get all orders for all accounts

##### Parameters

No parameters

##### Responses

**Success Response (200)**: A List of orders for the specified account or if its not mentioned, for all the linked accounts, matching the provided input parameters.

**Schema on success**:
```json
[
    {
        "session": "NORMAL",
        "duration": "DAY",
        "orderType": "MARKET",
        "cancelTime": "2025-11-08T22:40:38.934Z",
        "complexOrderStrategyType": "NONE",
        "quantity": 0,
        "filledQuantity": 0,
        "remainingQuantity": 0,
        "requestedDestination": "INET",
        "destinationLinkName": "string",
        "releaseTime": "2025-11-08T22:40:38.934Z",
        "stopPrice": 0,
        "stopPriceLinkBasis": "MANUAL",
        "stopPriceLinkType": "VALUE",
        "stopPriceOffset": 0,
        "stopType": "STANDARD",
        "priceLinkBasis": "MANUAL",
        "priceLinkType": "VALUE",
        "price": 0,
        "taxLotMethod": "FIFO",
        "orderLegCollection": [
            {
                "orderLegType": "EQUITY",
                "legId": 0,
                "instrument": {
                    "cusip": "string",
                    "symbol": "string",
                    "description": "string",
                    "instrumentId": 0,
                    "netChange": 0,
                    "type": "SWEEP_VEHICLE"
                },
                "instruction": "BUY",
                "positionEffect": "OPENING",
                "quantity": 0,
                "quantityType": "ALL_SHARES",
                "divCapGains": "REINVEST",
                "toSymbol": "string"
            }
        ],
        "activationPrice": 0,
        "specialInstruction": "ALL_OR_NONE",
        "orderStrategyType": "SINGLE",
        "orderId": 0,
        "cancelable": false,
        "editable": false,
        "status": "AWAITING_PARENT_ORDER",
        "enteredTime": "2025-11-08T22:40:38.934Z",
        "closeTime": "2025-11-08T22:40:38.934Z",
        "tag": "string",
        "accountNumber": 0,
        "orderActivityCollection": [
            {
                "activityType": "EXECUTION",
                "executionType": "FILL",
                "quantity": 0,
                "orderRemainingQuantity": 0,
                "executionLegs": [
                    {
                        "legId": 0,
                        "price": 0,
                        "quantity": 0,
                        "mismarkedQuantity": 0,
                        "instrumentId": 0,
                        "time": "2025-11-08T22:40:38.934Z"
                    }
                ]
            }
        ],
        "replacingOrderCollection": [
            "string"
        ],
        "childOrderStrategies": [
            "string"
        ],
        "statusDescription": "string"
    }
]
```

**Headers**:
- `Schwab-Client-CorrelId`: Correlation Id. Auto generated (string)


#### **POST** `/accounts/{accountNumber}/previewOrder`

Preview an order for a specific account.

##### Parameters

No parameters

##### Request body

**application/json**

The Order Object.

**Example Value**

```json
{
    "orderId": 0,
    "orderStrategy": {
        "accountNumber": "string",
        "advancedOrderType": "NONE",
        "closeTime": "2025-11-08T22:40:38.951Z",
        "enteredTime": "2025-11-08T22:40:38.951Z",
        "orderBalance": {
            "orderValue": 0,
            "projectedAvailableFund": 0,
            "projectedBuyingPower": 0,
            "projectedCommission": 0
        },
        "orderStrategyType": "SINGLE",
        "orderVersion": 0,
        "session": "NORMAL",
        "status": "AWAITING_PARENT_ORDER",
        "allOrNone": true,
        "discretionary": true,
        "duration": "DAY",
        "filledQuantity": 0,
        "orderType": "MARKET",
        "orderValue": 0,
        "price": 0,
        "quantity": 0,
        "remainingQuantity": 0,
        "sellNonMarginableFirst": true,
        "settlementInstruction": "REGULAR",
        "strategy": "NONE",
        "amountIndicator": "DOLLARS",
        "orderLegs": [
            {
                "askPrice": 0,
                "bidPrice": 0,
                "lastPrice": 0,
                "markPrice": 0,
                "projectedCommission": 0,
                "quantity": 0,
                "finalSymbol": "string",
                "legId": 0,
                "assetType": "EQUITY",
                "instruction": "BUY"
            }
        ]
    },
    "orderValidationResult": {
        "alerts": [
            {
                "validationRuleName": "string",
                "message": "string",
                "activityMessage": "string",
                "originalSeverity": "ACCEPT",
                "overrideName": "string",
                "overrideSeverity": "ACCEPT"
            }
        ],
        "accepts": [
            {
                "validationRuleName": "string",
                "message": "string",
                "activityMessage": "string",
                "originalSeverity": "ACCEPT",
                "overrideName": "string",
                "overrideSeverity": "ACCEPT"
            }
        ],
        "rejects": [
            {
                "validationRuleName": "string",
                "message": "string",
                "activityMessage": "string",
                "originalSeverity": "ACCEPT",
                "overrideName": "string",
                "overrideSeverity": "ACCEPT"
            }
        ],
        "reviews": [
            {
                "validationRuleName": "string",
                "message": "string",
                "activityMessage": "string",
                "originalSeverity": "ACCEPT",
                "overrideName": "string",
                "overrideSeverity": "ACCEPT"
            }
        ],
        "warns": [
            {
                "validationRuleName": "string",
                "message": "string",
                "activityMessage": "string",
                "originalSeverity": "ACCEPT",
                "overrideName": "string",
                "overrideSeverity": "ACCEPT"
            }
        ]
    },
    "commissionAndFee": {
        "commission": {
            "commissionLegs": [
                {
                    "commissionValues": [
                        {
                            "value": 0,
                            "type": "COMMISSION"
                        }
                    ]
                }
            ]
        },
        "fee": {
            "feeLegs": [
                {
                    "feeValues": [
                        {
                            "value": 0,
                            "type": "COMMISSION"
                        }
                    ]
                }
            ]
        },
        "trueCommission": {
            "commissionLegs": [
                {
                    "commissionValues": [
                        {
                            "value": 0,
                            "type": "COMMISSION"
                        }
                    ]
                }
            ]
        }
    }
}
```

##### Responses

**Success Response (200)**: An order object, matching the input parameters

**Schema on success**:
```json
{
    "orderId": 0,
    "orderStrategy": {
        "accountNumber": "string",
        "advancedOrderType": "NONE",
        "closeTime": "2025-11-08T22:40:38.954Z",
        "enteredTime": "2025-11-08T22:40:38.954Z",
        "orderBalance": {
            "orderValue": 0,
            "projectedAvailableFund": 0,
            "projectedBuyingPower": 0,
            "projectedCommission": 0
        },
        "orderStrategyType": "SINGLE",
        "orderVersion": 0,
        "session": "NORMAL",
        "status": "AWAITING_PARENT_ORDER",
        "allOrNone": true,
        "discretionary": true,
        "duration": "DAY",
        "filledQuantity": 0,
        "orderType": "MARKET",
        "orderValue": 0,
        "price": 0,
        "quantity": 0,
        "remainingQuantity": 0,
        "sellNonMarginableFirst": true,
        "settlementInstruction": "REGULAR",
        "strategy": "NONE",
        "amountIndicator": "DOLLARS",
        "orderLegs": [
            {
                "askPrice": 0,
                "bidPrice": 0,
                "lastPrice": 0,
                "markPrice": 0,
                "projectedCommission": 0,
                "quantity": 0,
                "finalSymbol": "string",
                "legId": 0,
                "assetType": "EQUITY",
                "instruction": "BUY"
            }
        ]
    },
    "orderValidationResult": {
        "alerts": [
            {
                "validationRuleName": "string",
                "message": "string",
                "activityMessage": "string",
                "originalSeverity": "ACCEPT",
                "overrideName": "string",
                "overrideSeverity": "ACCEPT"
            }
        ],
        "accepts": [
            {
                "validationRuleName": "string",
                "message": "string",
                "activityMessage": "string",
                "originalSeverity": "ACCEPT",
                "overrideName": "string",
                "overrideSeverity": "ACCEPT"
            }
        ],
        "rejects": [
            {
                "validationRuleName": "string",
                "message": "string",
                "activityMessage": "string",
                "originalSeverity": "ACCEPT",
                "overrideName": "string",
                "overrideSeverity": "ACCEPT"
            }
        ],
        "reviews": [
            {
                "validationRuleName": "string",
                "message": "string",
                "activityMessage": "string",
                "originalSeverity": "ACCEPT",
                "overrideName": "string",
                "overrideSeverity": "ACCEPT"
            }
        ],
        "warns": [
            {
                "validationRuleName": "string",
                "message": "string",
                "activityMessage": "string",
                "originalSeverity": "ACCEPT",
                "overrideName": "string",
                "overrideSeverity": "ACCEPT"
            }
        ]
    },
    "commissionAndFee": {
        "commission": {
            "commissionLegs": [
                {
                    "commissionValues": [
                        {
                            "value": 0,
                            "type": "COMMISSION"
                        }
                    ]
                }
            ]
        },
        "fee": {
            "feeLegs": [
                {
                    "feeValues": [
                        {
                            "value": 0,
                            "type": "COMMISSION"
                        }
                    ]
                }
            ]
        },
        "trueCommission": {
            "commissionLegs": [
                {
                    "commissionValues": [
                        {
                            "value": 0,
                            "type": "COMMISSION"
                        }
                    ]
                }
            ]
        }
    }
}
```

**Headers**:
- `Schwab-Client-CorrelId`: Correlation Id. Auto generated (string)



### Transactions

#### **GET**  `/accounts/{accountNumber}/transactions`

Get all transactions information for a specific account. All transactions for a specific account. Maximum number of transactions in response is 3000\. Maximum date range is 1 year.

##### Parameters

No parameters

##### Responses

**Success Response (200)**: A List of orders for the account, matching the provided input parameters

**Schema on success**:
```json
[
    {
        "activityId": 0,
        "time": "2025-11-08T22:40:38.974Z",
        "user": {
            "cdDomainId": "string",
            "login": "string",
            "type": "ADVISOR_USER",
            "userId": 0,
            "systemUserName": "string",
            "firstName": "string",
            "lastName": "string",
            "brokerRepCode": "string"
        },
        "description": "string",
        "accountNumber": "string",
        "type": "TRADE",
        "status": "VALID",
        "subAccount": "CASH",
        "tradeDate": "2025-11-08T22:40:38.974Z",
        "settlementDate": "2025-11-08T22:40:38.974Z",
        "positionId": 0,
        "orderId": 0,
        "netAmount": 0,
        "activityType": "ACTIVITY_CORRECTION",
        "transferItems": [
            {
                "instrument": {
                    "cusip": "string",
                    "symbol": "string",
                    "description": "string",
                    "instrumentId": 0,
                    "netChange": 0,
                    "type": "SWEEP_VEHICLE"
                },
                "amount": 0,
                "cost": 0,
                "price": 0,
                "feeType": "COMMISSION",
                "positionEffect": "OPENING"
            }
        ]
    }
]
```

**Headers**:
- `Schwab-Client-CorrelId`: Correlation Id. Auto generated (string)


#### **GET** `/accounts/{accountNumber}/transactions/{transactionId}`

Get specific transaction information for a specific account

##### Parameters

No parameters

#### Responses

**Success Response (200)**: A List of orders for the account, matching the provided input parameters

**Schema on success**:
```json
[
    {
        "activityId": 0,
        "time": "2025-11-08T22:40:38.987Z",
        "user": {
            "cdDomainId": "string",
            "login": "string",
            "type": "ADVISOR_USER",
            "userId": 0,
            "systemUserName": "string",
            "firstName": "string",
            "lastName": "string",
            "brokerRepCode": "string"
        },
        "description": "string",
        "accountNumber": "string",
        "type": "TRADE",
        "status": "VALID",
        "subAccount": "CASH",
        "tradeDate": "2025-11-08T22:40:38.987Z",
        "settlementDate": "2025-11-08T22:40:38.987Z",
        "positionId": 0,
        "orderId": 0,
        "netAmount": 0,
        "activityType": "ACTIVITY_CORRECTION",
        "transferItems": [
            {
                "instrument": {
                    "cusip": "string",
                    "symbol": "string",
                    "description": "string",
                    "instrumentId": 0,
                    "netChange": 0,
                    "type": "SWEEP_VEHICLE"
                },
                "amount": 0,
                "cost": 0,
                "price": 0,
                "feeType": "COMMISSION",
                "positionEffect": "OPENING"
            }
        ]
    }
]
```

**Headers**:
- `Schwab-Client-CorrelId`: Correlation Id. Auto generated (string)



### UserPreference

#### **GET** `/userPreference`

Get user preference information for the logged in user.

##### Parameters

No parameters

##### Responses

**Success Response (200)**: List of user preference values.

**Schema on success**:
```json
[
    {
        "accounts": [
            {
                "accountNumber": "string",
                "primaryAccount": false,
                "type": "string",
                "nickName": "string",
                "accountColor": "string",
                "displayAcctId": "string",
                "autoPositionEffect": false
            }
        ],
        "streamerInfo": [
            {
                "streamerSocketUrl": "string",
                "schwabClientCustomerId": "string",
                "schwabClientCorrelId": "string",
                "schwabClientChannel": "string",
                "schwabClientFunctionId": "string"
            }
        ],
        "offers": [
            {
                "level2Permissions": false,
                "mktDataPermission": "string"
            }
        ]
    }
]
```




## Market Data API

The Market Data API provides endpoints for quotes, price history, option chains, and instrument fundamentals. These endpoints are accessible under the `https://api.schwabapi.com/marketdata/v1` base URL.

## Quotes

### **GET** `/quotes`
### **GET** `/quotes/{symbol}`
Get a quote for a single symbol.

#### Parameters

* `symbol` (string (path), required): The symbol to retrieve a quote for.
* `fields` (string (query)): Optional. Comma-separated list of fields to return.

#### Responses
**Success Response (200)**: Quote data for the requested symbol.

**Schema on success**:
```json
{
    "AAPL": {
        "assetMainType": "EQUITY",
        "assetSubType": "COE",
        "quoteType": "NBBO",
        "realtime": true,
        "symbol": "AAPL",
        "quote": {
            "askPrice": 221.07,
            "askSize": 1,
            "bidPrice": 220.76,
            "bidSize": 3,
            "lastPrice": 220.97,
            "mark": 220.84,
            "netChange": -6.51,
            "netPercentChange": -2.86179005,
            "quoteTime": 1741737578221,
            "totalVolume": 76137410
        },
        "reference": {
            "cusip": "037833100",
            "description": "APPLE INC",
            "exchangeName": "NASDAQ"
        }
    }
}
```

Get quotes for a list of one or more symbols.

#### Parameters

* `symbols` (string (query), required): A comma-separated list of symbols (e.g., `AAPL,MSFT,GOOG`).
* `fields` (string (query)): Optional. Comma-separated list of fields to return. By default, returns all fields.
* `indicative` (boolean (query)): Optional. Include indicative quotes. Default `false`.

#### Responses
**Success Response (200)**: A map of symbols to their quote data.

**Schema on success**:
```json
{
    "AAPL": {
        "assetMainType": "EQUITY",
        "assetSubType": "COE",
        "quoteType": "NBBO",
        "realtime": true,
        "symbol": "AAPL",
        "quote": {
            "askPrice": 221.07,
            "askSize": 1,
            "bidPrice": 220.76,
            "bidSize": 3,
            "lastPrice": 220.97,
            "mark": 220.84,
            "netChange": -6.51,
            "netPercentChange": -2.86179005,
            "quoteTime": 1741737578221,
            "totalVolume": 76137410
        },
        "reference": {
            "cusip": "037833100",
            "description": "APPLE INC",
            "exchangeName": "NASDAQ"
        }
    }
}
```

## Price History

### **GET** `/pricehistory`
Get historical price data (candles) for a specific symbol.

#### Parameters

* `symbol` (string (query), required): The symbol to retrieve price history for.
* `periodType` (string (query), required): The type of period to show. Valid values are `day`, `month`, `year`, `ytd`. Default is `day`.
* `period` (integer (query), required): The number of periods to show. Example: For a `periodType` of `day`, a `period` of `10` shows 10 days of history.
* `frequencyType` (string (query), required): The type of frequency with which a new candle is formed. Valid values: `minute`, `daily`, `weekly`, `monthly`.
* `frequency` (integer (query), required): The number of the frequencyType to be included in each candle. Valid values for minute: 1, 5, 10, 15, 30. Valid values for daily/weekly/monthly: 1.
* `startDate` (integer (query), required): Start date as milliseconds since epoch.
* `endDate` (integer (query), required): End date as milliseconds since epoch.
* `needExtendedHoursData` (boolean (query), required): `true` to return extended hours data, `false` for regular market hours only. Default is `true`.

#### Responses
**Success Response (200)**: A valid price history response containing an array of candles.

**Schema on success**:
```json
{
    "candles": [
        {
            "open": 164.5,
            "high": 165.2,
            "low": 163.8,
            "close": 164.1,
            "volume": 1200000,
            "datetime": 1741737600000
        }
    ],
    "symbol": "GOOG",
    "empty": false
}
```

## Option Chains

### **GET** `/chains`
Get an option chain for a specific underlying symbol.

#### Parameters

* `symbol` (string (query), required): The underlying symbol.
* `contractType` (string (query), required): Type of contracts to return. Valid values are `CALL`, `PUT`, `ALL`. Default is `ALL`.
* `strikeCount` (integer (query), required): The number of strikes to return above and below the at-the-money price.
* `includeQuotes` (boolean (query), required): Include quotes for options in the option chain. Default is `false`.
* `strategy` (string (query), required): Default is `SINGLE`. Can be used to return chains formatted for complex strategies (e.g. `ANALYTICAL`, `COVERED`, `VERTICAL`, `CALENDAR`, `STRADDLE`, `STRANGLE`, `BUTTERFLY`, `CONDOR`, `DIAGONAL`, `COLLAR`, `ROLL`).
* `interval` (integer (query), required): Strike interval for spread strategy chains.
* `strike` (number (query), required): Return options only at this specific strike price.
* `range` (string (query), required): Return options for the given range. Valid values are `ITM` (in the money), `NTM` (near the money), `OTM` (out of the money), `SAK` (strikes above market), `SBK` (strikes below market), `SNK` (strikes near market), `ALL` (all strikes).
* `fromDate` (string (query), required): Only return expirations after this date. Format: `yyyy-MM-dd`.
* `toDate` (string (query), required): Only return expirations before this date. Format: `yyyy-MM-dd`.
* `volatility` (number (query), required): Volatility to use in calculations. Applies only to `ANALYTICAL` strategy chains.
* `underlyingPrice` (number (query), required): Underlying price to use in calculations. Applies only to `ANALYTICAL` strategy chains.
* `interestRate` (number (query), required): Interest rate to use in calculations. Applies only to `ANALYTICAL` strategy chains.
* `daysToExpiration` (integer (query), required): Days to expiration to use in calculations. Applies only to `ANALYTICAL` strategy chains.
* `expMonth` (string (query), required): Return only options expiring in the specified month. Valid values are `JAN`, `FEB`, `MAR`, `APR`, `MAY`, `JUN`, `JUL`, `AUG`, `SEP`, `OCT`, `NOV`, `DEC`, `ALL`. Default is `ALL`.
* `optionType` (string (query), required): Type of option contracts to return. Valid values: `S` (Standard), `NS` (Non-Standard), `ALL` (All options). Default is `ALL`.

#### Responses
**Success Response (200)**: A valid option chain response.

**Schema on success**:
```json
{
    "symbol": "GOOG",
    "status": "SUCCESS",
    "underlying": {
        "symbol": "GOOG",
        "description": "ALPHABET INC C",
        "last": 164.08,
        "mark": 163.94,
        "delayed": false
    },
    "strategy": "SINGLE",
    "isDelayed": false,
    "numberOfContracts": 132,
    "callExpDateMap": {
        "2025-04-25:29": {
            "95.0": [
                {
                    "putCall": "CALL",
                    "symbol": "GOOG 250425C00095000",
                    "description": "GOOG 04/25/2025 95.00 C",
                    "exchangeName": "OPR",
                    "bid": 67.45,
                    "ask": 71.45,
                    "last": 0.0,
                    "mark": 69.45,
                    "strikePrice": 95.0,
                    "expirationDate": "2025-04-25T20:00:00.000+00:00",
                    "daysToExpiration": 29,
                    "multiplier": 100.0,
                    "inTheMoney": true
                }
            ]
        }
    },
    "putExpDateMap": {
        "2025-04-25:29": {
            "95.0": [
                {
                    "putCall": "PUT",
                    "symbol": "GOOG 250425P00095000",
                    "description": "GOOG 04/25/2025 95.00 P",
                    "exchangeName": "OPR",
                    "bid": 0.0,
                    "ask": 0.02,
                    "last": 0.0,
                    "mark": 0.01,
                    "strikePrice": 95.0,
                    "expirationDate": "2025-04-25T20:00:00.000+00:00",
                    "daysToExpiration": 29,
                    "multiplier": 100.0,
                    "inTheMoney": false
                }
            ]
        }
    }
}
```

### **GET** `/expirationchain`
Get an option expiration chain for a specific underlying symbol.

#### Parameters

* `symbol` (string (query), required): The underlying symbol.

#### Responses
**Success Response (200)**: A list of option expirations.

**Schema on success**:
```json
{
    "expirationList": [
        {
            "expirationDate": "2025-03-14",
            "daysToExpiration": 2,
            "expirationType": "W",
            "settlementType": "P",
            "optionRoots": "GOOG",
            "standard": true
        }
    ]
}
```

## Movers

### **GET** `/movers/{symbol}`
Get movers for a specific index.

#### Parameters

* `symbol` (string (path), required): The index symbol. Valid values: `$DJI`, `$COMPX`, `$SPX`, `NYSE`, `NASDAQ`, `OTCBB`, `INDEX_ALL`, `EQUITY_ALL`, `OPTION_ALL`, `OPTION_PUT`, `OPTION_CALL`.
* `sort` (string (query), required): Sort direction. Valid values: `VOLUME`, `TRADES`, `PERCENT_CHANGE_UP`, `PERCENT_CHANGE_DOWN`.
* `frequency` (integer (query), required): Frequency in minutes. Valid values: `0`, `1`, `5`, `10`, `30`, `60`.

#### Responses
**Success Response (200)**: Top movers for the requested index.

**Schema on success**:
```json
{
    "screeners": [
        {
            "description": "NVIDIA CORP",
            "volume": 65591131,
            "lastPrice": 119.26,
            "netChange": 3.68,
            "marketShare": 13.84,
            "totalVolume": 473944635,
            "trades": 522082,
            "netPercentChange": 0.0318,
            "symbol": "NVDA"
        }
    ]
}
```

## Market Hours

### **GET** `/markets`
Get market hours for multiple markets.

#### Parameters

* `markets` (string (query), required): Comma-separated list of markets (e.g., `equity,option,future`).
* `date` (string (query), required): Date in `yyyy-MM-dd` format.

### **GET** `/markets/{market_id}`
Get market hours for a single market.

#### Parameters

* `market_id` (string (path), required): The market ID (e.g., `equity`, `option`).
* `date` (string (query), required): Date in `yyyy-MM-dd` format.

## Instruments

### **GET** `/instruments`
Search for instruments.

#### Parameters

* `symbol` (string (query), required): Symbol or search term.
* `projection` (string (query), required): Search type. Valid values: `symbol-search`, `symbol-regex`, `desc-search`, `desc-regex`, `search`, `fundamental`.

### **GET** `/instruments/{cusip_id}`
Get instrument details by CUSIP.

#### Parameters

* `cusip_id` (string (path), required): The CUSIP of the instrument.

#### Responses
**Success Response (200)**: Instrument data.

**Schema on success**:
```json
{
    "instruments": [
        {
            "cusip": "02079K107",
            "symbol": "GOOG",
            "description": "ALPHABET INC C",
            "exchange": "NASDAQ",
            "assetType": "EQUITY"
        }
    ]
}
```

## Streaming Data (WebSocket)

### Streaming Data Best Practices

The Schwab Streaming API utilizes WebSockets to deliver real-time data, which requires a completely different connection pattern than the standard HTTP REST endpoints.

### **1. Dedicated Login Sequence**
Before you can subscribe to any streams, you must perform a dedicated login via the WebSocket connection. The standard HTTP access token is not implicitly used by the socket; it must be authenticated as part of a `login` request payload over the stream.

### **2. Handler Registration Order**
When sending a subscription command (e.g., `ADD` or `SUBS` for a specific service), data may begin flowing back immediately. Ensure your client application registers its message handlers or callbacks *before* it dispatches the subscription request. If data arrives before handlers are ready, those messages will be permanently lost.

### **3. Changes vs. Whole Data Responses**
Different streams deliver data in different paradigms:
* **Changes (Deltas):** Streams like Level 1 Equities (`LEVELONE_EQUITIES`), Level 1 Options, and Level 1 Futures only stream *changes* to the data. The first message contains the full current state, but subsequent messages will only include the keys that have been updated. Your application must cache the initial state and merge these updates.
* **Whole Data:** Order Book streams (like `NYSE_BOOK` or `NASDAQ_BOOK`) stream the *entire* state of the book with every message, overwriting whatever you previously had.

### **4. Numerical Key Mappings**
Raw WebSocket payloads return data mapped to numeric keys to save bandwidth (e.g., instead of `"Bid Price": 217.86`, the payload will be `"1": 217.86`). Always ensure your client maintains a translation mapping to decode these integers back to their human-readable property names. Note that `"0"` is typically always reserved for the ticker symbol or stream key.

### Streaming Field Reference

The following tables map the numeric keys in streaming payloads to their human-readable field names.

### **LEVELONE_EQUITIES**
| Key | Field Name |
| :--- | :--- |
| 0 | Symbol |
| 1 | Bid Price |
| 2 | Ask Price |
| 3 | Last Price |
| 4 | Bid Size |
| 5 | Ask Size |
| 6 | Ask ID |
| 7 | Bid ID |
| 8 | Total Volume |
| 9 | Last Size |
| 10 | High Price |
| 11 | Low Price |
| 12 | Close Price |
| 13 | Exchange ID |
| 14 | Marginable |
| 15 | Description |
| 16 | Last ID |
| 17 | Open Price |
| 18 | Net Change |
| 19 | 52 Week High |
| 20 | 52 Week Low |
| 21 | PE Ratio |
| 22 | Annual Dividend Amount |
| 23 | Dividend Yield |
| 24 | NAV |
| 25 | Exchange Name |
| 26 | Dividend Date |
| 27 | Regular Market Quote |
| 28 | Regular Market Trade |
| 29 | Regular Market Last Price |
| 30 | Regular Market Last Size |
| 31 | Regular Market Net Change |
| 32 | Security Status |
| 33 | Mark Price |
| 34 | Quote Time in Long |
| 35 | Trade Time in Long |
| 36 | Regular Market Trade Time in Long |
| 37 | Bid Time |
| 38 | Ask Time |
| 39 | Ask MIC ID |
| 40 | Bid MIC ID |
| 41 | Last MIC ID |
| 42 | Net Percent Change |
| 43 | Regular Market Percent Change |
| 44 | Mark Price Net Change |
| 45 | Mark Price Percent Change |
| 46 | Hard to Borrow Quantity |
| 47 | Hard To Borrow Rate |
| 48 | Hard to Borrow |
| 49 | shortable |
| 50 | Post-Market Net Change |
| 51 | Post-Market Percent Change |

### **LEVELONE_OPTIONS**
| Key | Field Name |
| :--- | :--- |
| 0 | Symbol |
| 1 | Description |
| 2 | Bid Price |
| 3 | Ask Price |
| 4 | Last Price |
| 5 | High Price |
| 6 | Low Price |
| 7 | Close Price |
| 8 | Total Volume |
| 9 | Open Interest |
| 10 | Volatility |
| 11 | Money Intrinsic Value |
| 12 | Expiration Year |
| 13 | Multiplier |
| 14 | Digits |
| 15 | Open Price |
| 16 | Bid Size |
| 17 | Ask Size |
| 18 | Last Size |
| 19 | Net Change |
| 20 | Strike Price |
| 21 | Contract Type |
| 22 | Underlying |
| 23 | Expiration Month |
| 24 | Deliverables |
| 25 | Time Value |
| 26 | Expiration Day |
| 27 | Days to Expiration |
| 28 | Delta |
| 29 | Gamma |
| 30 | Theta |
| 31 | Vega |
| 32 | Rho |
| 33 | Security Status |
| 34 | Theoretical Option Value |
| 35 | Underlying Price |
| 36 | UV Expiration Type |
| 37 | Mark Price |
| 38 | Quote Time in Long |
| 39 | Trade Time in Long |
| 40 | Exchange |
| 41 | Exchange Name |
| 42 | Last Trading Day |
| 43 | Settlement Type |
| 44 | Net Percent Change |
| 45 | Mark Price Net Change |
| 46 | Mark Price Percent Change |
| 47 | Implied Yield |
| 48 | isPennyPilot |
| 49 | Option Root |
| 50 | 52 Week High |
| 51 | 52 Week Low |
| 52 | Indicative Ask Price |
| 53 | Indicative Bid Price |
| 54 | Indicative Quote Time |
| 55 | Exercise Type |

### **CHART_EQUITY**
| Key | Field Name |
| :--- | :--- |
| 0 | Key |
| 1 | Sequence |
| 2 | Open Price |
| 3 | High Price |
| 4 | Low Price |
| 5 | Close Price |
| 6 | Volume |
| 7 | Chart Time |
| 8 | Chart Day |

### **ACCT_ACTIVITY**
| Key | Field Name |
| :--- | :--- |
| seq | Sequence |
| key | Key |
| 1 | Account |
| 2 | Message Type |
| 3 | Message Data |


## Common HTTP Error Responses

The following error responses are common across most API endpoints. They typically return a standard `ServiceError` JSON schema (`{ "message": "string", "errors": [ "string" ] }`) with the `Schwab-Client-CorrelID` header.

* **400**: An error message indicating the validation problem with the request.
* **401**: An error message indicating either authorization token is invalid or there are no accounts the caller is allowed to view or use for trading that are registered with the provided third party application
* **403**: An error message indicating the caller is forbidden from accessing this service
* **404**: An error message indicating the resource is not found
* **500**: An error message indicating there was an unexpected server error
* **503**: An error message indicating server has a temporary problem responding

# Schemas

```typescript

export interface AccountNumberHash {
 accountNumber?: string;
 hashValue?: string;
}

/** The market session during which the order trade should be executed. */
export type session = 'NORMAL' /* Normal market hours, from 9:30am to 4:00pm Eastern. */ | 'AM' /* Premarket session, from 8:00am to 9:30am Eastern. */ | 'PM' /* After-market session, from 4:00pm to 8:00pm Eastern. */ | 'SEAMLESS' /* Orders are active during all trading sessions except the overnight session. This is the union of NORMAL, AM, and PM. */;

/** Length of time over which the trade will be active. */
export type duration = 'DAY';

/** Type of order to place. */
export type orderType = 'MARKET' /* Execute the order immediately at the best-available price. More Info < */ | 'LIMIT' /* Execute the order at your price or better. More info < */ | 'STOP' /* Wait until the price reaches the stop price, and then immediately place a market order. More Info < */ | 'STOP_LIMIT' /* Wait until the price reaches the stop price, and then immediately place a limit order at the specified price. More Info < */ | 'TRAILING_STOP' /* Similar to STOP, except if the price moves in your favor, the stop price is adjusted in that direction. Places a market order if the stop condition is met. More info < */ | 'CABINET' | 'NON_MARKETABLE' | 'MARKET_ON_CLOSE' /* Place the order at the closing price immediately upon market close. More info <>__ */ | 'EXERCISE' /* Exercise an option. */ | 'TRAILING_STOP_LIMIT' /* Similar to STOP_LIMIT, except if the price moves in your favor, the stop price is adjusted in that direction. Places a limit order at the specified price if the stop condition is met. More info < */ | 'NET_DEBIT' /* Place an order for an options spread resulting in a net debit. More info < whats-difference-between-credit-spread-and-debt-spread.asp>__ */ | 'NET_CREDIT' /* Place an order for an options spread resulting in a net credit. More info < whats-difference-between-credit-spread-and-debt-spread.asp>__ */ | 'NET_ZERO' /* Place an order for an options spread resulting in neither a credit nor a debit. More info < whats-difference-between-credit-spread-and-debt-spread.asp>__ */ | 'LIMIT_ON_CLOSE' | 'UNKNOWN';

/** Same as orderType, but does not have UNKNOWN since this type is not allowed as an input Type of order to place. */
export type orderTypeRequest = 'MARKET' /* Execute the order immediately at the best-available price. More Info < */ | 'LIMIT' /* Execute the order at your price or better. More info < */ | 'STOP' /* Wait until the price reaches the stop price, and then immediately place a market order. More Info < */ | 'STOP_LIMIT' /* Wait until the price reaches the stop price, and then immediately place a limit order at the specified price. More Info < */ | 'TRAILING_STOP' /* Similar to STOP, except if the price moves in your favor, the stop price is adjusted in that direction. Places a market order if the stop condition is met. More info < */ | 'CABINET' | 'NON_MARKETABLE' | 'MARKET_ON_CLOSE' /* Place the order at the closing price immediately upon market close. More info <>__ */ | 'EXERCISE' /* Exercise an option. */ | 'TRAILING_STOP_LIMIT' /* Similar to STOP_LIMIT, except if the price moves in your favor, the stop price is adjusted in that direction. Places a limit order at the specified price if the stop condition is met. More info < */ | 'NET_DEBIT' /* Place an order for an options spread resulting in a net debit. More info < whats-difference-between-credit-spread-and-debt-spread.asp>__ */ | 'NET_CREDIT' /* Place an order for an options spread resulting in a net credit. More info < whats-difference-between-credit-spread-and-debt-spread.asp>__ */ | 'NET_ZERO' /* Place an order for an options spread resulting in neither a credit nor a debit. More info < whats-difference-between-credit-spread-and-debt-spread.asp>__ */ | 'LIMIT_ON_CLOSE';

/** Explicit order strategies for executing multi-leg options orders. */
export type complexOrderStrategyType = 'NONE' /* No complex order strategy. This is the default. */ | 'COVERED' /* Covered call < selling-covered-call-options-strategy-income-hedging-15135>__ */ | 'VERTICAL' /* Vertical spread < vertical-credit-spreads-high-probability-15846>__ */ | 'BACK_RATIO' /* Ratio backspread < pricey-stocks-ratio-spreads-15306>__ */ | 'CALENDAR' /* Calendar spread < calendar-spreads-trading-primer-15095>__ */ | 'DIAGONAL' /* Diagonal spread < love-your-diagonal-spread-15030>__ */ | 'STRADDLE' /* Straddle spread < straddle-strangle-option-volatility-16208>__ */ | 'STRANGLE' /* Strandle spread < straddle-strangle-option-volatility-16208>__ */ | 'COLLAR_SYNTHETIC' | 'BUTTERFLY' /* Butterfly spread < butterfly-spread-options-15976>__ */ | 'CONDOR' /* Condor spread < condorspread.asp>__ */ | 'IRON_CONDOR' /* Iron condor spread < iron-condor-options-spread-your-trading-wings-15948>__ */ | 'VERTICAL_ROLL' /* Roll a vertical spread < exit-winning-losing-trades-16685>__ */ | 'COLLAR_WITH_STOCK' /* Collar strategy < stock-hedge-options-collars-15529>__ */ | 'DOUBLE_DIAGONAL' /* Double diagonal spread < the-ultimate-guide-to-double-diagonal-spreads/>__ */ | 'UNBALANCED_BUTTERFLY' /* Unbalanced butterfy spread < trading/unbalanced-butterfly-strong-directional-bias-15913>__ */ | 'UNBALANCED_CONDOR' | 'UNBALANCED_IRON_CONDOR' | 'UNBALANCED_VERTICAL_ROLL' | 'MUTUAL_FUND_SWAP' /* Mutual fund swap */ | 'CUSTOM' /* A custom multi-leg order strategy. */;

/** By default, Schwab sends trades to whichever exchange provides the best price. This field allows you to request a destination exchange for your trade, although whether your order is actually executed there is up to Schwab. Destinations for when you want to request a specific destination for your order. */
export type requestedDestination = 'INET' | 'ECN_ARCA' | 'CBOE' | 'AMEX' | 'PHLX' | 'ISE' | 'BOX' | 'NYSE' | 'NASDAQ' | 'BATS' | 'C2' | 'AUTO';

export type stopPriceLinkBasis = 'MANUAL' | 'BASE' | 'TRIGGER' | 'LAST' | 'BID' | 'ASK' | 'ASK_BID' | 'MARK' | 'AVERAGE';

export type stopPriceLinkType = 'VALUE' | 'PERCENT' | 'TICK';

export type stopType = 'STANDARD' | 'BID' | 'ASK' | 'LAST' | 'MARK';

export type priceLinkBasis = 'MANUAL' | 'BASE' | 'TRIGGER' | 'LAST' | 'BID' | 'ASK' | 'ASK_BID' | 'MARK' | 'AVERAGE';

export type priceLinkType = 'VALUE' | 'PERCENT' | 'TICK';

export type taxLotMethod = 'FIFO' | 'LIFO' | 'HIGH_COST' | 'LOW_COST' | 'AVERAGE_COST' | 'SPECIFIC_LOT' | 'LOSS_HARVESTER';

/** Special instruction for trades. */
export type specialInstruction = 'ALL_OR_NONE' /* Disallow partial order execution. More info < */ | 'DO_NOT_REDUCE' /* Do not reduce order size in response to cash dividends. More info < */ | 'ALL_OR_NONE_DO_NOT_REDUCE' /* Combination of ALL_OR_NONE and DO_NOT_REDUCE. */;

/** Rules for composite orders. */
export type orderStrategyType = 'SINGLE' /* No chaining, only a single order is submitted */ | 'CANCEL' | 'RECALL' | 'PAIR' | 'FLATTEN' | 'TWO_DAY_SWAP' | 'BLAST_ALL' | 'OCO' /* Execution of one order cancels the other */ | 'TRIGGER' /* Execution of one order triggers placement of the other */;

/** Order statuses passed to :meth:`get_orders_for_account` and
 :meth:`get_orders_for_all_linked_accounts` */
export type status = 'AWAITING_PARENT_ORDER' | 'AWAITING_CONDITION' | 'AWAITING_STOP_CONDITION' | 'AWAITING_MANUAL_REVIEW' | 'ACCEPTED' | 'AWAITING_UR_OUT' | 'PENDING_ACTIVATION' | 'QUEUED' | 'WORKING' | 'REJECTED' | 'PENDING_CANCEL' | 'CANCELED' | 'PENDING_REPLACE' | 'REPLACED' | 'FILLED' | 'EXPIRED' | 'NEW' | 'AWAITING_RELEASE_TIME' | 'PENDING_ACKNOWLEDGEMENT' | 'PENDING_RECALL' | 'UNKNOWN';

export type amountIndicator = 'DOLLARS' | 'SHARES' | 'ALL_SHARES' | 'PERCENTAGE' | 'UNKNOWN';

export type settlementInstruction = 'REGULAR' | 'CASH' | 'NEXT_DAY' | 'UNKNOWN';

export interface OrderStrategy {
 accountNumber?: string;
 advancedOrderType?: 'NONE' | 'OTO' | 'OCO' | 'OTOCO' | 'OT2OCO' | 'OT3OCO' | 'BLAST_ALL' | 'OTA' | 'PAIR';
 closeTime?: string;
 enteredTime?: string;
 orderBalance?: OrderBalance;
 orderStrategyType?: orderStrategyType;
 orderVersion?: number;
 session?: session;
 /** Restrict query to orders with this status. See :class:`Order.Status` for options. */
 status?: apiOrderStatus;
 allOrNone?: boolean;
 discretionary?: boolean;
 duration?: duration;
 filledQuantity?: number;
 orderType?: orderType;
 orderValue?: number;
 price?: number;
 /** Number of contracts for the order */
 quantity?: number;
 remainingQuantity?: number;
 sellNonMarginableFirst?: boolean;
 settlementInstruction?: settlementInstruction;
 /** If passed, returns a Strategy Chain. See :class:`Options.Strategy` for choices. */
 strategy?: complexOrderStrategyType;
 amountIndicator?: amountIndicator;
 orderLegs?: OrderLeg[];
}

export interface OrderLeg {
 askPrice?: number;
 bidPrice?: number;
 lastPrice?: number;
 markPrice?: number;
 projectedCommission?: number;
 /** Number of contracts for the order */
 quantity?: number;
 finalSymbol?: string;
 legId?: number;
 assetType?: assetType;
 /** Instruction for the leg. See :class:`~schwab.orders.common.OptionInstruction` for valid options. */
 instruction?: instruction;
}

export interface OrderBalance {
 orderValue?: number;
 projectedAvailableFund?: number;
 projectedBuyingPower?: number;
 projectedCommission?: number;
}

export interface OrderValidationResult {
 alerts?: OrderValidationDetail[];
 accepts?: OrderValidationDetail[];
 rejects?: OrderValidationDetail[];
 reviews?: OrderValidationDetail[];
 warns?: OrderValidationDetail[];
}

export interface OrderValidationDetail {
 validationRuleName?: string;
 message?: string;
 activityMessage?: string;
 originalSeverity?: APIRuleAction;
 overrideName?: string;
 overrideSeverity?: APIRuleAction;
}

/** string Enum: [ ACCEPT, ALERT, REJECT, REVIEW, UNKNOWN ] */
export type APIRuleAction = 'ACCEPT' | 'ALERT' | 'REJECT' | 'REVIEW' | 'UNKNOWN';

export interface CommissionAndFee {
 commission?: Commission;
 fee?: Fees;
 trueCommission?: Commission;
}

export interface Commission {
 commissionLegs?: any;
}

export interface CommissionLeg {
 commissionValues?: any;
}

export interface CommissionValue {
 value?: number;
 type?: FeeType;
}

export interface Fees {
 feeLegs?: any;
}

export interface FeeLeg {
 feeValues?: any;
}

export interface FeeValue {
 value?: number;
 type?: FeeType;
}

export type FeeType = 'COMMISSION' | 'SEC_FEE' | 'STR_FEE' | 'R_FEE' | 'CDSC_FEE' | 'OPT_REG_FEE' | 'ADDITIONAL_FEE' | 'MISCELLANEOUS_FEE' | 'FTT' | 'FUTURES_CLEARING_FEE' | 'FUTURES_DESK_OFFICE_FEE' | 'FUTURES_EXCHANGE_FEE' | 'FUTURES_GLOBEX_FEE' | 'FUTURES_NFA_FEE' | 'FUTURES_PIT_BROKERAGE_FEE' | 'FUTURES_TRANSACTION_FEE' | 'LOW_PROCEEDS_COMMISSION' | 'BASE_CHARGE' | 'GENERAL_CHARGE' | 'GST_FEE' | 'TAF_FEE' | 'INDEX_OPTION_FEE' | 'TEFRA_TAX' | 'STATE_TAX' | 'UNKNOWN';

export interface Account {
 securitiesAccount?: SecuritiesAccount;
}

export interface DateParam {
 /** Date for which to return market hours. Accepts values up to one year from today. Accepts ``datetime.date``. */
 date?: string;
}

export interface Order {
 session?: session;
 duration?: duration;
 orderType?: orderType;
 cancelTime?: string;
 complexOrderStrategyType?: complexOrderStrategyType;
 /** Number of contracts for the order */
 quantity?: number;
 filledQuantity?: number;
 remainingQuantity?: number;
 requestedDestination?: requestedDestination;
 destinationLinkName?: string;
 releaseTime?: string;
 stopPrice?: number;
 stopPriceLinkBasis?: stopPriceLinkBasis;
 stopPriceLinkType?: stopPriceLinkType;
 stopPriceOffset?: number;
 stopType?: stopType;
 priceLinkBasis?: priceLinkBasis;
 priceLinkType?: priceLinkType;
 price?: number;
 taxLotMethod?: taxLotMethod;
 orderLegCollection?: any;
 activationPrice?: number;
 specialInstruction?: specialInstruction;
 orderStrategyType?: orderStrategyType;
 orderId?: number;
 cancelable?: boolean;
 editable?: boolean;
 /** Restrict query to orders with this status. See :class:`Order.Status` for options. */
 status?: status;
 enteredTime?: string;
 closeTime?: string;
 tag?: string;
 accountNumber?: number;
 orderActivityCollection?: any;
 replacingOrderCollection?: OrderedMap[];
 childOrderStrategies?: any;
 statusDescription?: string;
}

export interface OrderRequest {
 session?: session;
 duration?: duration;
 orderType?: orderTypeRequest;
 cancelTime?: string;
 complexOrderStrategyType?: complexOrderStrategyType;
 /** Number of contracts for the order */
 quantity?: number;
 filledQuantity?: number;
 remainingQuantity?: number;
 destinationLinkName?: string;
 releaseTime?: string;
 stopPrice?: number;
 stopPriceLinkBasis?: stopPriceLinkBasis;
 stopPriceLinkType?: stopPriceLinkType;
 stopPriceOffset?: number;
 stopType?: stopType;
 priceLinkBasis?: priceLinkBasis;
 priceLinkType?: priceLinkType;
 price?: number;
 taxLotMethod?: taxLotMethod;
 orderLegCollection?: any;
 activationPrice?: number;
 specialInstruction?: specialInstruction;
 orderStrategyType?: orderStrategyType;
 orderId?: number;
 cancelable?: boolean;
 editable?: boolean;
 /** Restrict query to orders with this status. See :class:`Order.Status` for options. */
 status?: status;
 enteredTime?: string;
 closeTime?: string;
 accountNumber?: number;
 orderActivityCollection?: any;
 replacingOrderCollection?: OrderedMap[];
 childOrderStrategies?: OrderedMap[];
 statusDescription?: string;
}

export interface PreviewOrder {
 orderId?: number;
 orderStrategy?: OrderStrategy;
 orderValidationResult?: OrderValidationResult;
 commissionAndFee?: CommissionAndFee;
}

export interface OrderActivity {
 activityType?: 'EXECUTION' | 'ORDER_ACTION';
 executionType?: 'FILL';
 /** Number of contracts for the order */
 quantity?: number;
 orderRemainingQuantity?: number;
 executionLegs?: any;
}

export interface ExecutionLeg {
 legId?: number;
 price?: number;
 /** Number of contracts for the order */
 quantity?: number;
 mismarkedQuantity?: number;
 instrumentId?: number;
 time?: string;
}

export interface Position {
 shortQuantity?: number;
 averagePrice?: number;
 currentDayProfitLoss?: number;
 currentDayProfitLossPercentage?: number;
 longQuantity?: number;
 settledLongQuantity?: number;
 settledShortQuantity?: number;
 agedQuantity?: number;
 instrument?: AccountsInstrument;
 marketValue?: number;
 maintenanceRequirement?: number;
 averageLongPrice?: number;
 averageShortPrice?: number;
 taxLotAverageLongPrice?: number;
 taxLotAverageShortPrice?: number;
 longOpenProfitLoss?: number;
 shortOpenProfitLoss?: number;
 previousSessionLongQuantity?: number;
 previousSessionShortQuantity?: number;
 currentDayCost?: number;
}

export interface ServiceError {
 message?: string;
 errors?: string[];
}

export interface OrderLegCollection {
 orderLegType?: 'EQUITY' | 'OPTION' | 'INDEX' | 'MUTUAL_FUND' | 'CASH_EQUIVALENT' | 'FIXED_INCOME' | 'CURRENCY' | 'COLLECTIVE_INVESTMENT';
 legId?: number;
 instrument?: AccountsInstrument;
 /** Instruction for the leg. See :class:`~schwab.orders.common.OptionInstruction` for valid options. */
 instruction?: instruction;
 positionEffect?: 'OPENING' | 'CLOSING' | 'AUTOMATIC';
 /** Number of contracts for the order */
 quantity?: number;
 quantityType?: 'ALL_SHARES' | 'DOLLARS' | 'SHARES';
 divCapGains?: 'REINVEST' | 'PAYOUT';
 toSymbol?: string;
}

export type SecuritiesAccount = MarginAccount | CashAccount;

export interface SecuritiesAccountBase {
 type?: 'CASH' | 'MARGIN';
 accountNumber?: string;
 roundTrips?: number;
 isDayTrader?: boolean;
 isClosingOnlyRestricted?: boolean;
 pfcbFlag?: boolean;
 positions?: any;
}

export interface MarginAccount {
 type?: 'CASH' | 'MARGIN';
 accountNumber?: string;
 roundTrips?: number;
 isDayTrader?: boolean;
 isClosingOnlyRestricted?: boolean;
 pfcbFlag?: boolean;
 positions?: any;
 initialBalances?: MarginInitialBalance;
 currentBalances?: MarginBalance;
 projectedBalances?: MarginBalance;
}

export interface MarginInitialBalance {
 accruedInterest?: number;
 availableFundsNonMarginableTrade?: number;
 bondValue?: number;
 buyingPower?: number;
 cashBalance?: number;
 cashAvailableForTrading?: number;
 cashReceipts?: number;
 dayTradingBuyingPower?: number;
 dayTradingBuyingPowerCall?: number;
 dayTradingEquityCall?: number;
 equity?: number;
 equityPercentage?: number;
 liquidationValue?: number;
 longMarginValue?: number;
 longOptionMarketValue?: number;
 longStockValue?: number;
 maintenanceCall?: number;
 maintenanceRequirement?: number;
 margin?: number;
 marginEquity?: number;
 moneyMarketFund?: number;
 mutualFundValue?: number;
 regTCall?: number;
 shortMarginValue?: number;
 shortOptionMarketValue?: number;
 shortStockValue?: number;
 totalCash?: number;
 isInCall?: number;
 unsettledCash?: number;
 pendingDeposits?: number;
 marginBalance?: number;
 shortBalance?: number;
 accountValue?: number;
}

export interface MarginBalance {
 availableFunds?: number;
 availableFundsNonMarginableTrade?: number;
 buyingPower?: number;
 buyingPowerNonMarginableTrade?: number;
 dayTradingBuyingPower?: number;
 dayTradingBuyingPowerCall?: number;
 equity?: number;
 equityPercentage?: number;
 longMarginValue?: number;
 maintenanceCall?: number;
 maintenanceRequirement?: number;
 marginBalance?: number;
 regTCall?: number;
 shortBalance?: number;
 shortMarginValue?: number;
 sma?: number;
 isInCall?: number;
 stockBuyingPower?: number;
 optionBuyingPower?: number;
}

export interface CashAccount {
 type?: 'CASH' | 'MARGIN';
 accountNumber?: string;
 roundTrips?: number;
 isDayTrader?: boolean;
 isClosingOnlyRestricted?: boolean;
 pfcbFlag?: boolean;
 positions?: any;
 initialBalances?: CashInitialBalance;
 currentBalances?: CashBalance;
 projectedBalances?: CashBalance;
}

export interface CashInitialBalance {
 accruedInterest?: number;
 cashAvailableForTrading?: number;
 cashAvailableForWithdrawal?: number;
 cashBalance?: number;
 bondValue?: number;
 cashReceipts?: number;
 liquidationValue?: number;
 longOptionMarketValue?: number;
 longStockValue?: number;
 moneyMarketFund?: number;
 mutualFundValue?: number;
 shortOptionMarketValue?: number;
 shortStockValue?: number;
 isInCall?: number;
 unsettledCash?: number;
 cashDebitCallValue?: number;
 pendingDeposits?: number;
 accountValue?: number;
}

export interface CashBalance {
 cashAvailableForTrading?: number;
 cashAvailableForWithdrawal?: number;
 cashCall?: number;
 longNonMarginableMarketValue?: number;
 totalCash?: number;
 cashDebitCallValue?: number;
 unsettledCash?: number;
}

export interface TransactionBaseInstrument {
 assetType: 'EQUITY' | 'OPTION' | 'INDEX' | 'MUTUAL_FUND' | 'CASH_EQUIVALENT' | 'FIXED_INCOME' | 'CURRENCY' | 'COLLECTIVE_INVESTMENT';
 /** String representing CUSIP of instrument for which to fetch data. Note leading zeroes must be preserved. */
 cusip?: string;
 /** For ``FUNDAMENTAL`` projection, the symbol for which to get fundamentals. For other projections, a search term. See below for details. */
 symbol?: string;
 description?: string;
 instrumentId?: number;
 netChange?: number;
}

export interface AccountsBaseInstrument {
 assetType: 'EQUITY' | 'OPTION' | 'INDEX' | 'MUTUAL_FUND' | 'CASH_EQUIVALENT' | 'FIXED_INCOME' | 'CURRENCY' | 'COLLECTIVE_INVESTMENT';
 /** String representing CUSIP of instrument for which to fetch data. Note leading zeroes must be preserved. */
 cusip?: string;
 /** For ``FUNDAMENTAL`` projection, the symbol for which to get fundamentals. For other projections, a search term. See below for details. */
 symbol?: string;
 description?: string;
 instrumentId?: number;
 netChange?: number;
}

export interface AccountsInstrument {
 // oneOf: AccountOption | AccountCashEquivalent | AccountEquity | AccountFixedIncome | AccountMutualFund
}

export interface TransactionInstrument {
 // oneOf: Currency | Index | Future | Forex | TransactionEquity | TransactionOption | Product | TransactionMutualFund | TransactionCashEquivalent | CollectiveInvestment | TransactionFixedIncome
}

export interface TransactionCashEquivalent {
 assetType: 'EQUITY' | 'OPTION' | 'INDEX' | 'MUTUAL_FUND' | 'CASH_EQUIVALENT' | 'FIXED_INCOME' | 'CURRENCY' | 'COLLECTIVE_INVESTMENT';
 /** String representing CUSIP of instrument for which to fetch data. Note leading zeroes must be preserved. */
 cusip?: string;
 /** For ``FUNDAMENTAL`` projection, the symbol for which to get fundamentals. For other projections, a search term. See below for details. */
 symbol?: string;
 description?: string;
 instrumentId?: number;
 netChange?: number;
 type?: 'SWEEP_VEHICLE' | 'SAVINGS' | 'MONEY_MARKET_FUND' | 'UNKNOWN';
}

export interface CollectiveInvestment {
 assetType: 'EQUITY' | 'OPTION' | 'INDEX' | 'MUTUAL_FUND' | 'CASH_EQUIVALENT' | 'FIXED_INCOME' | 'CURRENCY' | 'COLLECTIVE_INVESTMENT';
 /** String representing CUSIP of instrument for which to fetch data. Note leading zeroes must be preserved. */
 cusip?: string;
 /** For ``FUNDAMENTAL`` projection, the symbol for which to get fundamentals. For other projections, a search term. See below for details. */
 symbol?: string;
 description?: string;
 instrumentId?: number;
 netChange?: number;
 type?: 'UNIT_INVESTMENT_TRUST' | 'EXCHANGE_TRADED_FUND' | 'CLOSED_END_FUND' | 'INDEX' | 'UNITS';
}

export type instruction = 'BUY_TO_OPEN';

export type assetType = 'EQUITY' | 'MUTUAL_FUND' | 'OPTION' | 'FUTURE' | 'FOREX' | 'INDEX' | 'CASH_EQUIVALENT' | 'FIXED_INCOME' | 'PRODUCT' | 'CURRENCY' | 'COLLECTIVE_INVESTMENT';

export interface Currency {
 assetType: 'EQUITY' | 'OPTION' | 'INDEX' | 'MUTUAL_FUND' | 'CASH_EQUIVALENT' | 'FIXED_INCOME' | 'CURRENCY' | 'COLLECTIVE_INVESTMENT';
 /** String representing CUSIP of instrument for which to fetch data. Note leading zeroes must be preserved. */
 cusip?: string;
 /** For ``FUNDAMENTAL`` projection, the symbol for which to get fundamentals. For other projections, a search term. See below for details. */
 symbol?: string;
 description?: string;
 instrumentId?: number;
 netChange?: number;
}

export interface TransactionEquity {
 assetType: 'EQUITY' | 'OPTION' | 'INDEX' | 'MUTUAL_FUND' | 'CASH_EQUIVALENT' | 'FIXED_INCOME' | 'CURRENCY' | 'COLLECTIVE_INVESTMENT';
 /** String representing CUSIP of instrument for which to fetch data. Note leading zeroes must be preserved. */
 cusip?: string;
 /** For ``FUNDAMENTAL`` projection, the symbol for which to get fundamentals. For other projections, a search term. See below for details. */
 symbol?: string;
 description?: string;
 instrumentId?: number;
 netChange?: number;
 type?: 'COMMON_STOCK' | 'PREFERRED_STOCK' | 'DEPOSITORY_RECEIPT' | 'PREFERRED_DEPOSITORY_RECEIPT' | 'RESTRICTED_STOCK' | 'COMPONENT_UNIT' | 'RIGHT' | 'WARRANT' | 'CONVERTIBLE_PREFERRED_STOCK' | 'CONVERTIBLE_STOCK' | 'LIMITED_PARTNERSHIP' | 'WHEN_ISSUED' | 'UNKNOWN';
}

export interface TransactionFixedIncome {
 assetType: 'EQUITY' | 'OPTION' | 'INDEX' | 'MUTUAL_FUND' | 'CASH_EQUIVALENT' | 'FIXED_INCOME' | 'CURRENCY' | 'COLLECTIVE_INVESTMENT';
 /** String representing CUSIP of instrument for which to fetch data. Note leading zeroes must be preserved. */
 cusip?: string;
 /** For ``FUNDAMENTAL`` projection, the symbol for which to get fundamentals. For other projections, a search term. See below for details. */
 symbol?: string;
 description?: string;
 instrumentId?: number;
 netChange?: number;
 type?: 'BOND_UNIT' | 'CERTIFICATE_OF_DEPOSIT' | 'CONVERTIBLE_BOND' | 'COLLATERALIZED_MORTGAGE_OBLIGATION' | 'CORPORATE_BOND' | 'GOVERNMENT_MORTGAGE' | 'GNMA_BONDS' | 'MUNICIPAL_ASSESSMENT_DISTRICT' | 'MUNICIPAL_BOND' | 'OTHER_GOVERNMENT' | 'SHORT_TERM_PAPER' | 'US_TREASURY_BOND' | 'US_TREASURY_BILL' | 'US_TREASURY_NOTE' | 'US_TREASURY_ZERO_COUPON' | 'AGENCY_BOND' | 'WHEN_AS_AND_IF_ISSUED_BOND' | 'ASSET_BACKED_SECURITY' | 'UNKNOWN';
 maturityDate?: string;
 factor?: number;
 multiplier?: number;
 variableRate?: number;
}

export interface Forex {
 assetType: 'EQUITY' | 'OPTION' | 'INDEX' | 'MUTUAL_FUND' | 'CASH_EQUIVALENT' | 'FIXED_INCOME' | 'CURRENCY' | 'COLLECTIVE_INVESTMENT';
 /** String representing CUSIP of instrument for which to fetch data. Note leading zeroes must be preserved. */
 cusip?: string;
 /** For ``FUNDAMENTAL`` projection, the symbol for which to get fundamentals. For other projections, a search term. See below for details. */
 symbol?: string;
 description?: string;
 instrumentId?: number;
 netChange?: number;
 type?: 'STANDARD' | 'NBBO' | 'UNKNOWN';
 baseCurrency?: Currency;
 counterCurrency?: Currency;
}

export interface Future {
 activeContract?: boolean;
 type?: 'STANDARD' | 'UNKNOWN';
 expirationDate?: string;
 lastTradingDate?: string;
 firstNoticeDate?: string;
 multiplier?: number;
 // oneOf: Currency | Index | Forex | TransactionEquity | TransactionOption | Product | TransactionMutualFund | TransactionCashEquivalent | CollectiveInvestment | TransactionFixedIncome
}

export interface Index {
 activeContract?: boolean;
 type?: 'BROAD_BASED' | 'NARROW_BASED' | 'UNKNOWN';
 // oneOf: Currency | Future | Forex | TransactionEquity | TransactionOption | Product | TransactionMutualFund | TransactionCashEquivalent | CollectiveInvestment | TransactionFixedIncome
}

export interface TransactionMutualFund {
 assetType: 'EQUITY' | 'OPTION' | 'INDEX' | 'MUTUAL_FUND' | 'CASH_EQUIVALENT' | 'FIXED_INCOME' | 'CURRENCY' | 'COLLECTIVE_INVESTMENT';
 /** String representing CUSIP of instrument for which to fetch data. Note leading zeroes must be preserved. */
 cusip?: string;
 /** For ``FUNDAMENTAL`` projection, the symbol for which to get fundamentals. For other projections, a search term. See below for details. */
 symbol?: string;
 description?: string;
 instrumentId?: any;
 netChange?: number;
 fundFamilyName?: string;
 fundFamilySymbol?: string;
 fundGroup?: string;
 type?: 'NOT_APPLICABLE' | 'OPEN_END_NON_TAXABLE' | 'OPEN_END_TAXABLE' | 'NO_LOAD_NON_TAXABLE' | 'NO_LOAD_TAXABLE' | 'UNKNOWN';
 exchangeCutoffTime?: string;
 purchaseCutoffTime?: string;
 redemptionCutoffTime?: string;
}

export interface TransactionOption {
 assetType: 'EQUITY' | 'OPTION' | 'INDEX' | 'MUTUAL_FUND' | 'CASH_EQUIVALENT' | 'FIXED_INCOME' | 'CURRENCY' | 'COLLECTIVE_INVESTMENT';
 /** String representing CUSIP of instrument for which to fetch data. Note leading zeroes must be preserved. */
 cusip?: string;
 /** For ``FUNDAMENTAL`` projection, the symbol for which to get fundamentals. For other projections, a search term. See below for details. */
 symbol?: string;
 description?: string;
 instrumentId?: number;
 netChange?: number;
 expirationDate?: string;
 optionDeliverables?: any;
 optionPremiumMultiplier?: number;
 putCall?: 'PUT' | 'CALL' | 'UNKNOWN';
 strikePrice?: number;
 type?: 'VANILLA' | 'BINARY' | 'BARRIER' | 'UNKNOWN';
 underlyingSymbol?: string;
 underlyingCusip?: string;
 deliverable?: TransactionInstrument;
}

export interface Product {
 assetType: 'EQUITY' | 'OPTION' | 'INDEX' | 'MUTUAL_FUND' | 'CASH_EQUIVALENT' | 'FIXED_INCOME' | 'CURRENCY' | 'COLLECTIVE_INVESTMENT';
 /** String representing CUSIP of instrument for which to fetch data. Note leading zeroes must be preserved. */
 cusip?: string;
 /** For ``FUNDAMENTAL`` projection, the symbol for which to get fundamentals. For other projections, a search term. See below for details. */
 symbol?: string;
 description?: string;
 instrumentId?: number;
 netChange?: number;
 type?: 'TBD' | 'UNKNOWN';
}

export interface AccountCashEquivalent {
 assetType: 'EQUITY' | 'OPTION' | 'INDEX' | 'MUTUAL_FUND' | 'CASH_EQUIVALENT' | 'FIXED_INCOME' | 'CURRENCY' | 'COLLECTIVE_INVESTMENT';
 /** String representing CUSIP of instrument for which to fetch data. Note leading zeroes must be preserved. */
 cusip?: string;
 /** For ``FUNDAMENTAL`` projection, the symbol for which to get fundamentals. For other projections, a search term. See below for details. */
 symbol?: string;
 description?: string;
 instrumentId?: number;
 netChange?: number;
 type?: 'SWEEP_VEHICLE' | 'SAVINGS' | 'MONEY_MARKET_FUND' | 'UNKNOWN';
}

export interface AccountEquity {
 assetType: 'EQUITY' | 'OPTION' | 'INDEX' | 'MUTUAL_FUND' | 'CASH_EQUIVALENT' | 'FIXED_INCOME' | 'CURRENCY' | 'COLLECTIVE_INVESTMENT';
 /** String representing CUSIP of instrument for which to fetch data. Note leading zeroes must be preserved. */
 cusip?: string;
 /** For ``FUNDAMENTAL`` projection, the symbol for which to get fundamentals. For other projections, a search term. See below for details. */
 symbol?: string;
 description?: string;
 instrumentId?: number;
 netChange?: number;
}

export interface AccountFixedIncome {
 assetType: 'EQUITY' | 'OPTION' | 'INDEX' | 'MUTUAL_FUND' | 'CASH_EQUIVALENT' | 'FIXED_INCOME' | 'CURRENCY' | 'COLLECTIVE_INVESTMENT';
 /** String representing CUSIP of instrument for which to fetch data. Note leading zeroes must be preserved. */
 cusip?: string;
 /** For ``FUNDAMENTAL`` projection, the symbol for which to get fundamentals. For other projections, a search term. See below for details. */
 symbol?: string;
 description?: string;
 instrumentId?: number;
 netChange?: number;
 maturityDate?: string;
 factor?: number;
 variableRate?: number;
}

export interface AccountMutualFund {
 assetType: 'EQUITY' | 'OPTION' | 'INDEX' | 'MUTUAL_FUND' | 'CASH_EQUIVALENT' | 'FIXED_INCOME' | 'CURRENCY' | 'COLLECTIVE_INVESTMENT';
 /** String representing CUSIP of instrument for which to fetch data. Note leading zeroes must be preserved. */
 cusip?: string;
 /** For ``FUNDAMENTAL`` projection, the symbol for which to get fundamentals. For other projections, a search term. See below for details. */
 symbol?: string;
 description?: string;
 instrumentId?: number;
 netChange?: number;
}

export interface AccountOption {
 assetType: 'EQUITY' | 'OPTION' | 'INDEX' | 'MUTUAL_FUND' | 'CASH_EQUIVALENT' | 'FIXED_INCOME' | 'CURRENCY' | 'COLLECTIVE_INVESTMENT';
 /** String representing CUSIP of instrument for which to fetch data. Note leading zeroes must be preserved. */
 cusip?: string;
 /** For ``FUNDAMENTAL`` projection, the symbol for which to get fundamentals. For other projections, a search term. See below for details. */
 symbol?: string;
 description?: string;
 instrumentId?: number;
 netChange?: number;
 optionDeliverables?: any;
 putCall?: 'PUT' | 'CALL' | 'UNKNOWN';
 optionMultiplier?: number;
 type?: 'VANILLA' | 'BINARY' | 'BARRIER' | 'UNKNOWN';
 underlyingSymbol?: string;
}

export interface AccountAPIOptionDeliverable {
 /** For ``FUNDAMENTAL`` projection, the symbol for which to get fundamentals. For other projections, a search term. See below for details. */
 symbol?: string;
 deliverableUnits?: number;
 apiCurrencyType?: 'USD' | 'CAD' | 'EUR' | 'JPY';
 assetType?: 'EQUITY' | 'MUTUAL_FUND' | 'OPTION' | 'FUTURE' | 'FOREX' | 'INDEX' | 'CASH_EQUIVALENT' | 'FIXED_INCOME' | 'PRODUCT' | 'CURRENCY' | 'COLLECTIVE_INVESTMENT';
}

export interface TransactionAPIOptionDeliverable {
 rootSymbol?: string;
 strikePercent?: number;
 deliverableNumber?: number;
 deliverableUnits?: number;
 deliverable?: TransactionInstrument;
 assetType?: assetType;
}

export type apiOrderStatus = 'AWAITING_PARENT_ORDER' | 'AWAITING_CONDITION' | 'AWAITING_STOP_CONDITION' | 'AWAITING_MANUAL_REVIEW' | 'ACCEPTED' | 'AWAITING_UR_OUT' | 'PENDING_ACTIVATION' | 'QUEUED' | 'WORKING' | 'REJECTED' | 'PENDING_CANCEL' | 'CANCELED' | 'PENDING_REPLACE' | 'REPLACED' | 'FILLED' | 'EXPIRED' | 'NEW' | 'AWAITING_RELEASE_TIME' | 'PENDING_ACKNOWLEDGEMENT' | 'PENDING_RECALL' | 'UNKNOWN';

/** string Enum: [ TRADE, RECEIVE_AND_DELIVER, DIVIDEND_OR_INTEREST, ACH_RECEIPT, ACH_DISBURSEMENT, CASH_RECEIPT, CASH_DISBURSEMENT, ELECTRONIC_FUND, WIRE_OUT, WIRE_IN, JOURNAL, MEMORANDUM, MARGIN_CALL, MONEY_MARKET, SMA_ADJUSTMENT ] */
export type TransactionType = 'TRADE' | 'RECEIVE_AND_DELIVER' | 'DIVIDEND_OR_INTEREST' | 'ACH_RECEIPT' | 'ACH_DISBURSEMENT' | 'CASH_RECEIPT' | 'CASH_DISBURSEMENT' | 'ELECTRONIC_FUND' | 'WIRE_OUT' | 'WIRE_IN' | 'JOURNAL' | 'MEMORANDUM' | 'MARGIN_CALL' | 'MONEY_MARKET' | 'SMA_ADJUSTMENT';

export interface Transaction {
 activityId?: number;
 time?: string;
 user?: UserDetails;
 description?: string;
 accountNumber?: string;
 type?: TransactionType;
 /** Restrict query to orders with this status. See :class:`Order.Status` for options. */
 status?: 'VALID' | 'INVALID' | 'PENDING' | 'UNKNOWN';
 subAccount?: 'CASH' | 'MARGIN' | 'SHORT' | 'DIV' | 'INCOME' | 'UNKNOWN';
 tradeDate?: string;
 settlementDate?: string;
 positionId?: number;
 orderId?: number;
 netAmount?: number;
 activityType?: 'ACTIVITY_CORRECTION' | 'EXECUTION' | 'ORDER_ACTION' | 'TRANSFER' | 'UNKNOWN';
 transferItems?: TransferItem[];
}

export interface UserDetails {
 cdDomainId?: string;
 login?: string;
 type?: 'ADVISOR_USER' | 'BROKER_USER' | 'CLIENT_USER' | 'SYSTEM_USER' | 'UNKNOWN';
 userId?: number;
 systemUserName?: string;
 firstName?: string;
 lastName?: string;
 brokerRepCode?: string;
}

export interface TransferItem {
 instrument?: TransactionInstrument;
 amount?: number;
 cost?: number;
 price?: number;
 feeType?: 'COMMISSION' | 'SEC_FEE' | 'STR_FEE' | 'R_FEE' | 'CDSC_FEE' | 'OPT_REG_FEE' | 'ADDITIONAL_FEE' | 'MISCELLANEOUS_FEE' | 'FUTURES_EXCHANGE_FEE' | 'LOW_PROCEEDS_COMMISSION' | 'BASE_CHARGE' | 'GENERAL_CHARGE' | 'GST_FEE' | 'TAF_FEE' | 'INDEX_OPTION_FEE' | 'UNKNOWN';
 positionEffect?: 'OPENING' | 'CLOSING' | 'AUTOMATIC' | 'UNKNOWN';
}

export interface UserPreference {
 accounts?: UserPreferenceAccount[];
 streamerInfo?: StreamerInfo[];
 offers?: Offer[];
}

export interface UserPreferenceAccount {
 accountNumber?: string;
 primaryAccount?: boolean;
 type?: string;
 nickName?: string;
 accountColor?: string;
 displayAcctId?: string;
 autoPositionEffect?: boolean;
}

export interface StreamerInfo {
 streamerSocketUrl?: string;
 schwabClientCustomerId?: string;
 schwabClientCorrelId?: string;
 schwabClientChannel?: string;
 schwabClientFunctionId?: string;
}

export interface Offer {
 level2Permissions?: boolean;
 mktDataPermission?: string;
}
```

