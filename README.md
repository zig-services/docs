# Integrating with ZIG

## Backend

The integration consists of three main points:

* **game discovery** You can query the zig service for a list of games that are
  currently registered for your tenant (thats you) and site (like sub-tenant).
  This list of games will include information such as the games _canonical name_,
  its display name, the base ticket price and more. This includes also your custom
  frontend url that you will need to pass to the frontend integration to actually
  load the game for a customer in an iframe.

* **buy phase** Once the customer decides to play a game, he sends a buy request
  to your backend. Your backend will need to validate the customers session
  and balance for the requested ticket. If everything is in order, a ticket is
  requested from the zig service and the customers balance is updated. The
  ticket needs to be proxied to the game.

* **settle phase** The player will now see the game playing in his browser.
  At the end of the sequence, the game will ask your backend to _settle_ the
  ticket. Settling a ticket allows you to book the players winnings.


### Communication with the zig service

You get an https address and an access token from zig that you need to use
to talk to the zig service. To authenticate yourself with the zig service, you
are required to send a `Authorization: Bearer $token` header containing
your token.

The responses of the zig service will be encoded as json. The responses
may contain additional fields to the ones that are documented here: _Please do
not use the values of those fields!_

### Game discovery

You can query the zig service for a list of games. To do so, you need to send
a `GET` request to `https://$zigService/tenants/$yourTenant/sites/$yourSite/games`.
This endpoint will return a list of `Game` instances containing the following
fields:

```java
class Game {
  String gameName;
  String displayName;

  // url pointing to the frontend resources
  String gameUrl;

  // describes if the game is active (can be played)
  // and visible (should be shown to the customer)
  boolean active;
  boolean visible;

  // base ticket price for quantity=1, bet factor=1
  MoneyAmount price;

  // list of values allowed for bet factor parameter.
  int[] allowedBetFactors;

  // maximum value for the quantity paramter
  int maximumQuantity;
}
```

The `MoneyAmount` type is defined with three fields containing the amount in minor
units, in major units (normally the same as `amountInMinor` scaled by 100) and the
currency as a three letter uppercase code like `EUR` or `GBP`.

```java
class MoneyAmount {
  int amountInMinor;
  float amountInMajor;
  String currency;
}
```

The response of the `games` endpoint should be cached in your platform for a few
seconds, but should be updated regulary to detect new games or updates of
existing games.

### Buy phase

Once the customer has choosen to play a game, the frontend will execute a call to your
backend on the path `/zig/games/$gameName/tickets:buy`. The game might pass a numeric
`betFactor` and `quantity` url parameter. Your platform needs to execute the following steps:

* Validating the customers request by checking if a game with this canonical
  game name exists. Using the game information fetched in the previous
  game discovery step, your backend can now calculate the ticket price by
  multiplying the ticket price from the game with the `betFactor` and `quantity`
  url parameter. If one of those parameters is not given, it is assumed to be one.

  You can now check the customers balance against the calculated ticket price. If the
  customers balance is less than the ticket price, you need to fail the request with
  an error type of `insufficiant-balance`.

  Finally you reserve the money on your customers balance.

* Next you call the zig service to get a ticket. You do a `POST` request to
  the endpoint `/tenants/$yourTenant/sites/$yourSite/games/$gameName/customers/$customerIdentifier/tickets`.

  The path contains a customer identifier. This identifier should identify a customer
  on your side. We do not require you to use your internal customer id. A better solution
  is to calculate a salted hash of your customers id, as that would still ensure uniqueness
  while not exposing your internal customer id.

  If the original request contained a `betFactor` or `quantity` url parameter, you need to pass
  those values on as `betFactor` and `quantity`. If the request is successful, the zig service
  will respond with a ticket for the user.

  The purchase was successful and the customers balance can now be reduced by the tickets price.
  To verify that your previous calculations were correct, you can check the `price` field in
  the received ticket. The backend should then serialize the ticket into your database and pass it as a response
  back to the user.

  The ticket format is described by this java object:

  ```java
  class Ticket {
    String id;
    String ticketNumber;

    // The amount of money the customer payed for this ticket.
    MoneyAmount price;

    // The winning class of the ticket. Use this to extract the winnings
    // of this ticket. If the winnings are zero this was a loosing bet.
    WinningClass winningClass;

    // The tickets scenario, not relevant for the backend.
    Scenario string;
  }

  class WinningClass {
    int number;
    MoneyAmount winnings;
  }
  ```

### Settle phase

When the player finishes the game, it will instruct the backend to settle the still open ticket by
sending a `POST` request to the path `/zig/games/$gameName/tickets:settle/$id`. The id field in the
url will have the value of the `id` field of the ticket to settle. Your backend needs to execute the
following steps:

* Settle the ticket on the zig service by sending a `POST` request to
  `/tenants/$yourTenant/sites/$yourSite/games/$gameName/customers/$customerIdentifier/tickets/$id/settle`.
  The call will mark the ticket as settled in the zig service.

* Book the customers winning to his account. The winnings are already included in the
  ticket you stored in your platforms database.


## Frontend

Checkout the documentation on [zig-js](https://zig-services.github.io/zig-js/#integrating-games-into-your-platform).
