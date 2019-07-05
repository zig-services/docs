# Integrating with ZIG RGS

## Overview

For you (the operator) to integrate ZIG into your platform, you are required to implement a secured json rest interface in your backend. The ZIG remote game service contacts this API to authenticate customers playing games on your platform and to execute financial transactions during gameplay.


### ZIG API

### Launch
Once a customer opens a game on your platform, the operator’s backend sends a launch-game request to the ZIG remote game service to begin a new game session. Such a request includes the player id and game name that the player wishes to launch. To support internationalization, the request must also include the customer’s locale (e.g. en_GB) and currency (e.g. EUR). Finally the operator need to pass an url endpoint that the remote game service will later contact to execute financial transactions for the customer.
The launch game request will return a session id (token) that from then on authenticates requests by the remote game service for this customer on the operator’s platform. It will be included in all further requests for the session that was just created.
Furthermore the response includes a opaque gameConfig data structure that needs to be passed to the ZIG javascript library in the frontend. The library will setup and display the game in an iframe inside the operator’s website.

### Operator API

### Auth
Once a player opens a game, an (optional) Authentication request will be sent to the operator in order to validate the Token of the player.

### Balance
The first request done by the remote game service is usual a request to check the customer’s balance. This is done again and again from time to time, e.g. after starting or finishing a game round to provide the customer with the most recent balance in the frontend.

### Debit and Credit
After loading the game the customer selects his stake and presses the play button. The remote game service initializes a new game round and needs to debit the customer’s account for the ticket price. It does so by triggering a debit transaction with the operator by including the session token created above, a unique transaction id and the debit amount. The operator must now process the transaction and debit the requested amount from the customer’s account.
Following the debit any potential winnings will be credited to the customer. The credit transaction looks is equal to a debit transaction, except that it books money onto the customer’s account balance.
The last debit or credit transaction during a game round will include a flag to signal the end of the game round. If the customer continues to play another game round, the remote game service will start the next game round by executing a debit transaction.
In case of errors, the remote game service will attempt to retry or cancel (void) the transaction.

### Void
Under circumstances the remote game service might need to cancel a previously executed transaction. In this case a void transaction request is sent to the operator, containing the unique id of the transaction to be canceled. Canceling a debit transaction refunds the customer with the amount previously debited, canceling a credit transaction must remove the previously booked winnings from the customer’s balance.
