# Socky API v0.5

This is draft for protocol of communication between Socky elements. It's open for discussion so please write every proposition under this message - this will be discussed and final version will be posted on github repository.

## Naming:

Socky specification defines 3 components:

- Server: main core waiting for connections from clients. Base implementation(in Ruby) will be handled [here](http://github.com/socky/socky-server-ruby).
- WebSocket Client: component allowing client to connect to Server. Base implementation(in JavaScript) will be handled [here](http://github.com/socky/socky-js). From here it will be called "WS-Client".
- Authentication Module: component allowing authenticate Client rights to connect to Server. Base implementation(in Ruby) will be handled [here](http://github.com/socky/socky-authenticator-ruby). From here it will be called "Authenticator".

Note that this split to 3 parts is used just to allow demonstrate protocol. Any part can be written in any language and, unless it create security risk, merged in one module.

## Additional naming:

- Connection: TCP socket between WS-Client and Server
- Application: Namespace of connections. Connections from one application can affect other connections from the same namespace, but no other connections.
  - Application name: Unique name by which application can be identified. Can contain only alphanumeric characters(case sensitive) plus dash and underscore.
- Channel: Group of connections. Every connection can have multiple channels.

## Starting notes:

All Socky components, if not merged together, should connect using JSON-encoded hashes. What it mean is that every component upon receiving data should decode it and threat as hash. If data type is not hash then such request should be ignored. In addition to this each part of specification allows for number of keys. If key is not mentioned in concrete part of specification then it should be ignored and not parsed or stored.

Communication between WS-Client and Server should use WebSocket protocol. Communication between WS-Client and Authenticator or Server and Authenticator(if it's not built in) should use HTTP requests.

Internal communication of merged modules will not be specified in this document, so can be implemented at the discretion of the programmer.

## Opening connection between WS-Client and Server:

If WS-Client want to connect to Server then it opens standard WebSocket connection. Except of server host and port there should be path provided.

Path is created by 2 parts: Socky backend namespace and target application name.

Default Socky backend namespace should be '/websocket', but there should be possibility to change that.

### Example:

WS-Client want to connect to example.com, port 8080, with default backend namespace and application named 'my_application'. Path will be:

    ws://example.com:8080/websocket/my_application

If application name will not be recognized by Server then it should send following hash and close connection afterwards:

    { event: 'socky:connection:error', reason: 'refused' }

If Server recognize application name then it allows connection and send following hash to WS-Client:

    { event: 'socky:connection:established', connection_id: '<connection_id>' }

Connection_id should be unique identifier of connection. It should contain from 1 to 20 characters, including only letters, numbers, dash and underscore. WS-Client should save that id - it will be required to further identifying of connection.

## Subscribing to channel:

After WS-Client receive connection_id it can subscribe to channel. This specification describes 3 kinds of channel:

- Public channel: Channel that anybody can join without authentication. This channel type is used at default.
- Private channel: Channel that need authentication by Authenticator. This channel is used if channel name has prefix 'private-'
- Presence channel: Channel that allow notification about joining and leaving connections. This channel require authentication and is used if channel name has prefix 'presence-'

Each channel name can contain from 1 to 30 characters, including only letters, numbers, dash and underscore.

## Subscribing to public channel:

Any connected WS-Client can subscribe to public channel. In order to do so WS-Client should send the following hash to Server:

    { event: 'socky:subscribe', channel: 'desired_channel' }

In return to such request Server should join WS-Client to channel and return:

    { event: 'socky:subscribe:success', channel: <requested_channel> }

If (for any reason) Server will not be able to join WS-Client to channel then it should return:

    { event: 'socky:subscribe:failure', channel: <requested_channel> }

## Subscribing to private channel:

Private channel is channel that name starts with 'private-'. So valid example will be 'private-channel' but not '_private-channel'.

Private channels require WS-Client to ask Authenticator for channel authentication token. In order to do so WS-Client should send POST request to Authenticator with following hash:

    { event: 'socky:subscribe', channel: 'private-desired_channel', connection_id: <connection_id> }

Request url should be configurable, but should default to '/socky/auth'.

If request return status other that 200 then authentication should be counted as unsuccessful. If status is equal 200 then request body should contain one hash - authentication data for channel(JSON-encoded). Way of generating channel authentication token will be described later.

Received authentication data should be sent by WS-Client to Server using following hash:

    { event: 'socky:subscribe', channel: 'private-desired_channel', auth: <authentication_token> }

Server should return as in public channel.

## Subscribing to presence channel:

Connection process of presence channel is similar to private channel. The only addition is that WS-Client is allowed to push his own data to client in 'data' string:

    { event: 'socky:subscribe', channel: 'presence-desired_channel', connection_id: <connection_id>, data: { some: 'data' } }

Authenticator will return auth data and provided user data in JSON format. This data should be passed to Server without further conversion:

    { event: 'socky:subscribe', channel: 'presence-desired_channel', auth: <authentication_data>, data: <user_data> }

If subscription is successful then subscribing WS-Client will receive subscription confirmation and members list attached:

    { event: 'socky_internal:subscribe:success', channel: <requested_channel>, members: <member_list> }

Member list is array of hashes containing connection\_ids and user data of each member:

    'members' => [
                   { connection_id: 'first_connection_id', data: <user_data> },
                   { connection_id: 'second_connection_id', data: <user_data> }
                 ]

Other members of presence channel should receive notification about new channel member:

    { event: 'socky:member:added', connection_id: <connection_id>, channel: <channel>, data: <user_data> }

Note that WS-Client sending user data to Authenticator send it as hash. Authenticator returns this data in JSON-encoded format and in that form should be pushed to Server. Server decode JSON and send both subscribing WS-Client and other WS-Clients data in hash format. This is required to preserve hash keys order both in Authenticator and Server for purpose of signing request.

## Unsubscribing from public and private channel:

To unsubscribe from public or private channel WS-Client need to send:

    { event: 'socky:unsubscribe', channel: 'desired_channel' }

In return it will receive from Server:

    { event: 'socky:unsubscribe:success', channel: <requested_channel> }

If unsubscribing was unsuccessful(i.e. wrong channel name or user wasn't connected to this channel) then Server should return:

    { event: 'socky:unsubscribe:failure', channel: <requested_channel> }

## Unsubscribing from presence channel:

Unsubscribing from presence channel looks like public and private channel, but after unsubscribing all other WS-Clients subscribed to it should receive notification about that. Notification will include channel and connection\_id, but data should be taken from earlier received subscribe method.

    { event: 'socky:member:removed', connection_id: <connection_id>, channel: <channel> }

Note that if WS-Client will disconnect from server then all channels should receive unsubscribe notification.

## Additional rights for channel:

When WS-Client is subscribing to private or presence channel, it can provide additional parameters changing its rights on channel. This parameters are:

- "read" - should WS-Client receive data sent by this channel? Default: true
- "write" - should WS-Client be able to send data to this channel? Default: false
- "hide" - should other members of presence channel be notified about this WS-Client?(only for presence channel) Default: false

Parameters that are not changed from default should be skipped for preserve of bandwidth. Parameters that are changed should have value in boolean format.

### Explanation:

This 3 rights allow greater control over channel. I.e. let's assume that someone want to implement clone of Skype. WS-Client should be able to both read and write to channel(chat with one specified person). So he will have rights { 'read' => true, 'write' => true }. Note that 'read' option can be skipped as it is at default. Now let's assume channel with contact list. This will be presence-channel. Some users want status "invisible" - you can see others, but they can't see you. This will be { 'read' => true, 'hide' => true }. Global notification system will be { 'read' => false, 'write' => true } - it will send messages to all, but it don't need to receive any.

### Example:

If WS-Client want to subscribe as read/write to private-channel it will send to Authenticator:

    { event: 'socky:subscribe', channel: 'private-desired_channel', connection_id: <connection_id>, write: true }

Note that 'true' is not string 'true' but boolean value true.

After receiving authentication token WS-Client will send to Server:

    { event: 'socky:subscribe', channel: 'presence-desired_channel', auth: <authentication_data>, write: true }

For obvious reason only presence-channel should accept right 'hide' - private channel should just ignore it.

### Implementation guidelines:

Authenticator should have option to enable and disable changing rights by WS-Client. This function should be disabled at default, so developer will not forget about checking if WS-Client are allowed to grant himself provided rights. If this option is disabled and WS-Client will try to change his rights then authentication should be aborted and non-200 status returned.

## Authentication of channel subscription request:

Authentication is done 2 times: first time on WS-Client side when it checks if he will be able to subscribe and receive authentication token. Second is on Server side, when Server confirms authentication token. It will be good idea to have Authenticator build in Server, as this will not compromise security, and will fasten authentication a lot.

## Generating channel authentication data for private channels:

Authentication data should look like

    <salt>:<signature>

where salt should be random string(base implementations will use MD5 hash).

Signature is a HMAC SHA256 hex digest. This is generated by signing the following string with you application secret key:

    <salt>:<connection_id>:<channel_name>:<rights>

Note that rights should be converted to 0-1 format with order 'read','write','hide'. So rights

    { read: true, write: false, hide: false }

will look like '100'. In private channel 'hide' will always be false, but it should still be included as '0'. Please remember that default value is '100' and params not provided will have this value.

### Example:

    salt = 'somerandomstring'
    connection_id = '1234ABCD'
    channel_name = 'some_channel'
    write = true

    string_to_sign = 'somerandomstring:1234ABCD:some_channel:110'

In Ruby signing will look like:

    require 'hmac-sha2'

    secret = 'application_secret_key'
    signature = HMAC::SHA256.hexdigest(secret, string_to_sign)
    # => 'ebb032a2b6814f305571cf08a212cde14d21abc1a3076a257223e808c675b579'

Final auth string will look like:

    auth = 'somerandomstring:ebb032a2b6814f305571cf08a212cde14d21abc1a3076a257223e808c675b579'

When asked by WS-Client, Authenticator should return JSON-encoded hash:

    { auth: 'somerandomstring:ebb032a2b6814f305571cf08a212cde14d21abc1a3076a257223e808c675b579' }

## Generating channel authentication data for presence channels:

Generating auth data for presence channels is similar to private channels, with following exceptions:

WS-Client is allowed to send additional 'data' parameter and it should be remembered in JSON-encoded form.

    user_data = JSON.generate( request['data'] )

This data should be included in string\_to\_sign:

    string_to_sign = '<salt>:<connection_id>:<channel_name>:<rights>:<user_data>'

When asked by WS-Client, Authenticator should return JSON-encoded hash:

    { auth: '<salt>:<signature>', data: '<user_data>' }

Note that in final string user\_data will be JSON-encoded two times.

## Sending data to other WS-Clients

Every WS-Client with 'write' right can send event to channel, and it will be propagated to other clients. To do so WS-Client need to send following request to server:

    { event: 'some_event', channel: 'private-channel' }

This will trigger 'some_event' event for all other clients on this channel. Note, that currently only sending to private and presence channels is supported, as public channels aren't validated. Also, event should not be sent to clients with 'read' right set to false.

Event name can be anything from 1 to 30 characters. The only restriction for event naming is that user should not be allowed to send events starting with "socky:", which is reserved for internal methods. If client or server receive event that have such prefix and it's not described in this document then this event should not be propagated. Otherwise it should be sent to every WS-Client connected to channel.

In addition to mentioned earlier 'event' and 'channel' keys client should be able to provide 'data' for other users. This data should be hash and should be sent to other clients without any conversion.

    { event: 'some_event', channel: 'private-channel', data: { some: 'data' } }
