# Checkout flow

The checkout flow described bellow enables third-party client applications to create and process payments in a browser or a native application
such as mobile application.


_Processing payments is available only for browsers with CORS support. There are no specific restrictions related to native applications._


##1. Roles

Client
> is the backend of a user facing application. It is responsible for the server side communicution about authorization, checkout creation and status retrieval 

User-Agent or App
> is the UI of the client application that might be rendered in a browser. It collects user input about payment instrument (card) and requests completion of the payment passing passing card details.

Authorization server
> the server that provides authentication, checkout creation/processing and details about the state of checkouts

##2. Flow

`
         +-----------------+                          +---------------------+
         |                 <---------(E1) ------------+                     |
         |  User-Agent or  |                          |  Authorization      |
         |  Native App     |                          |  Server             |
         |                 +---------(E)-------------->                     |
         +---^---------+---+                          +------^-------^--^-^-+
             |         |                                     |       |  | |
             |         |                                     |       |  | |
            (D)       (D1)                                   |       |  | |
             |         |                                     |       |  | |
             |         |                                     |       |  | |
             |         |                                     |       |  | |
         +---+---------v---+                                 |       |  | |
         |                 +----------(F)--------------------+       |  | |
         |  Client         |                                         |  | |
         |                 |                                         |  | |
         |                 |                                         |  | |
         |                 +----------------------------(C)----------+  | |
         |                 +--------------(B)---------------------------+ |
         |                 +-(A)------------------------------------------+
         +-----------------+
`

##3. Flow steps

###3.1 (A) Authentication
The first step for a client is to authenticate with client credentials. After successful authentication it obtains an access token that must be used in checkout related requests. Example:

_Request_


    POST /v0.1/oauth
    Headers: Content-Type: application/json
    
    {
        "client_id": "...",
        "client_secret": "..."
    }

_Response_

    HTTP Status 200
    
    {
        "access_token": "....."
    }    

###3.2 (B) Create checkout
Checkout is created in a server to server comunication between the Client and the Authorization Server for an amount that can't be altered after its creation. The request body should include the following attributes:

**amount**
> (mandatory) the checkout payable amount

**currency**
> (mandatory) the checkout currency. Should be the same as the payee's account currency. Supported currencies are EUR, GBP, BRL, PLN, CHF, SEK, USD

**pay\_to\_email**
> (mandatory) the email of the payee. Should be a registered SumUp account that is allowed to receive payments

**checkout_reference**
> (mandatory) the client should use this parameter to pass its own identifier for the checkout that can be used later for reconciliation purposes - match specific checkout

**description**
> (optional) description of the checkout

Example:

_Request_


    POST /v0.1/checkout
    Headers: Content-Type: application/json
             Authorization: Bearer {access_token}
    
    {
        "amount": 10.5,
        "currency": "EUR",
        "pay_to_email": "payee@mail.com",
        "checkout_reference": "my-inique-identifier"
    }

_Response_

    HTTP Status 200
    
    {
        "id": "80e5e401-a503-4333-a446-6f190c08d617",
        "checkout_reference": "my-inique-identifier",
        "amount": 10.5,
        "currency": "EUR",
        "pay_to_email": "payee@mail.com",
        "status": "PENDING",
        "date": "2016-01-01T01:00:00.251Z"
    }    

Possible response status values can be PAID | PENDING | FAILED

###3.3 (C) Obtain payment authorization code
In order to complete a checkout in a browser an authorization should be used. The authorization code is a token valid for one request. Example of Client request

    POST /one-time-tokens
    Headers: Authorization: Bearer {access_token}
             X-Sumup-Allow-Origin: my.domain.com
    

The header X-Sumup-Allow-Origin should be set to the Client domain in order to enable CORS requests needed to complete the checkout.

The server response to the above request is `{"otpToken": ...}`

###3.4 (D) Expose payment info to the browser
The Client is exposing the authorization code and checkout id so that the checkout can be completed in the user-agent.


###3.5 (E) Complete payment
The checkout can be completed in the user-agent by a simple ajax PUT request as

    
    PUT /v0.1/checkouts/:checkoutId?otp={auth_code}
    
    {
      "payment_type":"card",
        "card": {
            "cvv": "...",
            "expiry_month": "01",
            "expiry_year": "2016",
            "number": ".......",
          "name":"...."
        }
    }
    

The path parameter `checkoutId` is set to the checkout id obtained in step 3.2 (B) ant the query parameter `otp` is set to the authorization code obtained in 3.3 (C)

The above request will return a checkout object (see 3.2 for example).

###3.6 Additional steps (D1,F)
After processing a checkout the client can check its state by a GET request. This call will return a checkout object as described in 3.2 

Example request:

    
    GET /v0.1/checkouts/:checkoutId
    Headers: Authorization: Bearer {access_token}
    
