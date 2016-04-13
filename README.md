# Checkout flow

The checkout flow described below enables third-party applications to create and process payments in a browser or a native application
such as mobile app.

_Processing payments is available only for browsers with CORS support. There are no specific restrictions related to native applications._

##1. Roles

Client
> The **user interface** that might be rendered in a browser or in a mobile app. It passes user-provided card details to the Server, which will in turn send a request to process payments through SumUp.

Server
> The **backend** that serves the client application. It is responsible for the server-side communicution with SumUp with regards to authorization, creating checkouts, and retrieving the status of a checkout.

SumUp
> The **SumUp's server** that provides authentication, checkout creation/processing, and details about the state of checkouts.

##2. Flow

`
         +-----------------+                          +------------------------+
         |                 <---------(E1) ------------+                        |
         |  Client         |                          |  SumUp                 |
         |  (Web|Mobile)   |                          |  (Auth/Payment Server) |
         |                 +---------(E)-------------->                        |
         +---^---------+---+                          +------^-------^--^-^----+
             |         |                                     |       |  | |
             |         |                                     |       |  | |
            (D)       (D1)                                   |       |  | |
             |         |                                     |       |  | |
             |         |                                     |       |  | |
             |         |                                     |       |  | |
         +---+---------v---+                                 |       |  | |
         |                 +----------(F)--------------------+       |  | |
         |  Server         |                                         |  | |
         |  (Your          |                                         |  | |
         |  Backend)       |                                         |  | |
         |                 +----------------------------(C)----------+  | |
         |                 +--------------(B)---------------------------+ |
         |                 +-(A)------------------------------------------+
         +-----------------+
`

##3. Flow steps

###3.1 (A) Get an access token from SumUp
First, your Server needs to authenticate with SumUp. Send a request to SumUp with your client credentials (obtained on the SumUp Dashboard). SumUp will respond with an `access_token` that must be included in subsequent requests relating to this checkout. For example:

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

###3.2 (B) Create a checkout
A new checkout is created in a server-to-server communication between your Server and SumUp. The amount of the checkout cannot be altered after it's been created. The body of the request should include the following parameters:

**amount**
> (required) The payable amount.

**currency**
> (required) The checkout currency. Should be the same as the payee's account currency (based on the registration country). Supported currencies include EUR, GBP, BRL, PLN, CHF, SEK, USD.

**pay\_to\_email**
> (required) The email of the payee. Should correspond to a registered SumUp account that is allowed to receive payments.

**checkout_reference**
> (required) The Server should use this parameter to pass its own identifier for the checkout, which can be used later for reconciliation purposes - match specific checkout (?)

**description**
> (optional) A description of the checkout.

Example:

_Request_


    POST /v0.1/checkout
    Headers: Content-Type: application/json
             Authorization: Bearer {access_token}
    
    {
        "amount": 10.5,
        "currency": "EUR",
        "pay_to_email": "payee@mail.com",
        "checkout_reference": "my-unique-identifier"
    }

_Response_

    HTTP Status 200
    
    {
        "id": "80e5e401-a503-4333-a446-6f190c08d617",
        "checkout_reference": "my-unique-identifier",
        "amount": 10.5,
        "currency": "EUR",
        "pay_to_email": "payee@mail.com",
        "status": "PENDING",
        "date": "2016-01-01T01:00:00.251Z"
    }    

Possible response status values can be PAID | PENDING | FAILED.

###3.3 (C) Obtain a payment authorization code
In order to complete a checkout in a browser, an authorization code must accompany the request. This code is valid for a single request. An example of a request to obtain this code between your Server and SumUp:

    POST /one-time-tokens
    Headers: Authorization: Bearer {access_token}
             X-Sumup-Allow-Origin: my.domain.com
    

The header X-Sumup-Allow-Origin should be set to the Client domain in order to enable CORS requests needed to complete the checkout.

The server response to the above request is `{ "otpToken": "..." }`

###3.4 (D) Expose payment info to the browser
Your Server then needs to expose the authorization code and checkout id to the Client, so that the checkout can be completed based on an action that happens on the Client (for example, clicking a "Pay Now" button).


###3.5 (E) Complete payment
After creating the checkout, the Client can finalize it by passing the payment detils with a simple ajax PUT request, like:

    
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
    

The URI parameter `checkoutId` should be the `id` obtained in step 3.2 (B) as part of the checkout response from SumUp, for example:

    /v0.1/checkouts/80e5e401-a503-4333-a446-6f190c08d617?otp={auth_code}

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

And the query parameter `otp` should be the authorization code obtained in 3.3 (C).

After you make this request, SumUp will respond with a checkout object (see 3.2 for an example).

###3.6 Additional steps (D1,F)
After processing a checkout, the client can check its state via GET request. This call will return a checkout object as described in 3.2. 

Example request:

    
    GET /v0.1/checkouts/:checkoutId
    Headers: Authorization: Bearer {access_token}
    
