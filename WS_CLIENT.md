# Socky Client API v0.5

This is draft of specification of callbacks given by WS-Client on specific actions. Most of that is described in SPECIFICATION.md, but it's short review for end users.

## socky:connection:established

Called when client connect to server

### Params:

- connection_id: unique id of user

## socky:connection:broken

Called when client disconnect from server. After this callback client should reconnect automaticaly. This callback is received only if connection was established earlier.

## socky:connection:error

Called when client is unable to make connection. This usually mean that server is down or refuse connection.

### Params:

- reason: reason of rejecting connection. Currently supported reasons:
  - down: server is down
  - refused: server refused application name

## Example:

When server refuse your application name then this will be called:

    { event: 'socky:connection:error', reason: 'refused' }

## socky:subscribe:success

This is called when client successfuly connect to channel.

### Params:

- channel: name of channel
- members: array of clients already connected to channel(only for presence channels)

## socky:subscribe:failure

Called when client wasn't able to connect to channel. This should be called if eaither authenticator or serve refuse connection.

### Params:

- channen: name of channel

## socky:unsubscribe:success

Called when client unsubscribed from channel.

### Params:

- channel: name of channel

## socky:unsubscribe:failure

Called when client wasn't able to unsubscribe from channel. Usually this mean that he wasn't connected earlier.

### Params:

- channel: name of channel

## socky:member:added

Called when new client join presence channel

### Params:

- channel: name of channel
- connection_id: unique id of client
- data: user specific data

## socky:member:removed

Called when client connected to presence channel leaves it

### Params:

- channel: name of channel
- connection_id: unique id of client

## Other

Some clients are allowed to send data to channel. All other members of channel will receive callback with specified by client name.

### Params:

- channel: name of channel
- data: user specified data

### Example:

    { event: 'some_event', channel: 'private-channel', data: { my: 'data' } }

This could be handled as:

    private-channel.bind('some_event', function(data) {
      alert(data.my);
    });
