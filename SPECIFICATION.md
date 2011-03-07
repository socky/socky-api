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
  - Application name: Unique name by which application can be identified
- Channel: Group of connections. Every connection can have multiple channels.
  - Public channel: Channel that anybody can join without authentication. This channel type is used at default.
  - Private channel: Channel that need authentication by client. This channel is used if channel name has prefix 'private-'
- User: Group of connections. Every connection can have only one user.
