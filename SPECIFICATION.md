# Socky API v0.5

This is draft for protocol of communication between Socky elements. It's open for discussion so please write every proposition under this message - this will be discussed and final version will be posted on github repository.

## Naming:

Socky is created from 3 components: Socky server, Socky JS client library and Socky web-server client library.

- Socky server will wait for connections from clients. Base impementation will be handled [here](http://github.com/socky/socky-server-ruby). From here it will be called "server".
- Socky JS client is used to connect browser to server and receive messages. Base implementation will be handled [here](http://github.com/socky/socky-js). From here it will be called "browser".
- Socky web-server client is used to authenticate browser and send messages to server. Base implementation will be handled [here](http://github.com/socky/socky-client-ruby). From here it will be called "client".

Note that this split to 3 parts is used just to allow demonstrate protocol. Any part can be written in any language and name can be then misleading(i.e. iOS library that will connect and just receive messages will be "browser"). Both "browser" and "client" can be also merged into one part if this connection won't create security risk.

## Additional naming:

- Connection: TCP socket between browser and server
- Application: Namespace of connections. Connections from one application can affect other connections from the same namespace, but no other connections.
  - Application name: Unique name by which application can be identified. Can contain only alphanumeric characters(case sensitive) plus dash and underscore.
- Channel: Group of connections. Every connection can have multiple channels.
  - Public channel: Channel that anybody can join without authentication. This channel type is used at default.
  - Private channel: Channel that need authentication by client. This channel is used if channel name has prefix 'private-'
  - Presence channel: Channel that allow notification about joining and leaving connections. This channel require authentication and is used if channel name has prefix 'presence-'
- User: Group of connections. Every connection can have only one user.

## Starting notes

All data sent over WebSocket protocol should be encoded using JSON. Every JSON will be hash in root, and each action specify exactly which keys can be used. If any component find in this hash keys that are outside of specification then it should drop that key and send data without it included.

## Connecting browser to server

If browser want to connect to server then it opens WebSocket connection. Except of server host and port there should be path provided.

Path is created by 2 parts: Socky backend namespace and target application name.

Default Socky backend namespace should be '/websocket', but there should be possibility to change that.

### Example:

User want to connect to example.com, port 8080, with default backend namespace and application named 'my_application'. Path will be:

    ws://example.com:8080/websocket/my_application

If application name will not be recognized then server should send following hash and close connection afterwards:

    { 'event' => 'socky:error:unknow_application' }

If server recognize application name then it allows connection and send following hash to browser:

    { 'event' => 'socky:connection_established', 'connection_id' => '<connection_id>' }

Connection_id should be unique identifier of connection. It should contain from 1 to 20 alphanumeric characters encoded as string. Browser should save that id - it will be required to further identifying of connection.

## Connecting to public channel

Any connected browser can subscribe to public channel. In order to do so browser should send to server following hash:

    { 'event' => 'socky:subscribe', 'channel' => 'desired_channel' }

In return to such request server should join browser to channel and return:

    { 'event' => 'socky_internal:subscription_successful', 'channel' => <requested_channel> }

If (for any reason) server will not be able to join browser to channel then it should return:

    { 'event' => 'socky_internal:subscription_unsuccessful', 'channel' => <requested_channel> }

## Connecting to private channel

Private channel is channel that name starts with 'private-'. So valid example will be 'private-channel' but not '_private-channel'.

Private channels require browser to ask client for channel authentication token. In order to do so browser should send POST request to client with following hash:

    { 'event' => 'socky:subscribe', 'channel' => 'private-desired_channel', 'connection_id' => <connection_id> }

Request url should be configurable, but should default to '/socky/auth'.

If request return status other that 200 then authentication should be counted as unsuccessful. If status is equal 200 then request body should contain one string - authentication data for channel. Way of generating channel authentication data will be described later.

Received authentication data should be sent to server using following hash:

    { 'event' => 'socky:subscribe', 'channel' => 'private-desired_channel', 'auth' => <authentication_data> }

Server should return as in public channel.

## Connecting to presence channel

Connection process of presence channel is similar to private channel. The only addition is that browser is allowed to push his own data to client in 'data' string:

    { 'event' => 'socky:subscribe', 'channel' => 'presence-desired_channel', 'connection_id' => <connection_id>, 'data' => { 'some' => 'data' } }

Client will return auth data and provided user data in JSON format. This data should be passed to server without further conversion:

    { 'event' => 'socky:subscribe', 'channel' => 'presence-desired_channel', 'auth' => <authentication_data>, 'data' => <user_data> }

If subscription is successful then subscribing browser will receive subscription confirmation and members list attached:

    { 'event' => 'socky_internal:subscription_successful', 'channel' => <requested_channel>, 'members' => <member_list> }
    
Member list is array of hashes containing connection\_ids and user data of each member:

    'members' => [
                   { 'connection_id' => 'first_connection_id', 'data' => <user_data> },
                   { 'connection_id' => 'second_connection_id', 'data' => <user_data> }
                 ]

Other members of presence channel should receive notification about new channel member:

    { 'event' => 'socky_internal:member_added', 'connection_id' => <connection_id>, 'channel' => <channel>, 'data' => <user_data> }

Note that browser sending user data to client send it as hash. Client returns this data in JSON-encoded format and in that form should be pushed to server. Server decode JSON and send both subscribing browser and other browser data in hash format. This is required to preserve hash keys order both in client and server for purpose of signing request.

## Disconnecting from presence channel

After disconnecting from presence channel all other browser subscribed to it should receive notification about that. Notification will include channel and connection\_id, but data should be taken from earlier received subscribe method.

    { 'event' => 'socky_internal:member_removed', 'connection_id' => <connection_id>, 'channel' => <channel> }

## Server authentication of private and presence channel join request

Server should implement generator of channel authentication data - the same as in client. Upon receiving subscribe request to private or presence channel it should generate authentication data and check if it match received one - if yes then it should join browser to channel and return 'subscription\_successful' event. Otherwise it should return 'subscription\_unsuccessful' event.

## Generating channel authentication data for private channels

Authentication data should look like

    <salt>:<signature>

where salt should be random string(base implementations will use MD5 hash).

Signature is a HMAC SHA256 hex digest. This is generated by signing the following string with you application secret key:

    <salt>:<connection_id>:<channel_name>

### Example:

    salt = 'somerandomstring'
    connection_id = '1234ABCD'
    channel_name = 'some_channel'
    
    string_to_sign = 'somerandomstring:1234ABCD:some_channel'

In Ruby signing will look like:

    require 'hmac-sha2'
    
    secret = 'application_secret_key'
    signature = HMAC::SHA256.hexdigest(secret, string_to_sign)
    # => '18f7322656afd59f69e002b717664bdb1aecd28194cdfa0cd4d90a17bf38f6f2'

Final auth string will look like:

    auth = 'somerandomstring:18f7322656afd59f69e002b717664bdb1aecd28194cdfa0cd4d90a17bf38f6f2'

When asked by browser, client should return JSON-encoded hash:

    { 'auth' => 'somerandomstring:18f7322656afd59f69e002b717664bdb1aecd28194cdfa0cd4d90a17bf38f6f2' }

## Generating channel authentication data for private channels

Generating auth data for private channels is similar to private channels, with following exceptions:

Browser is allowed to send additional 'data' parameter and it should be remembered in JSON-encoded form.

    user_data = JSON.generate( request['data'] )

This data should be included in string\_to\_sign:

    string_to_sign = '<salt>:<connection_id>:<channel_name>:<user_data>'

When asked by browser, client should return JSON-encoded hash:

    { 'auth' => '<salt>:<signature>', 'data' => '<user_data>' }

Note that in final string user\_data will be JSON-encoded two times.