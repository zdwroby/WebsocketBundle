<!-- vim: set ft=markdown tw=79 sw=4 ts=4 et : -->
# VarspoolWebsocketBundle

Alpha stability. Provides websocket services, including an in-built server,
multiplexing, semantic configuration.

## Installation

VarspoolWebsocketBundle depends on:

* Wrench (formerly for php-websocket), version 2.0.0-beta ([varspool/Wrench](https://github.com/varspool/Wrench))
  * A simple WebSockets library for PHP 5.3, providing PHP support for WebSocket
     clients and servers, using streams.

And, of course, Symfony2. Mostly, the bundle is a light compatibility layer
over WebSocket 2.0 that allows it to be used with the Service Container.

### Configuring dependencies

#### Composer

If you're using Composer, add the following packages to your project's
requires:

```json
{
    "require": {
        "varspool/websocket-bundle": "dev-master",
        "wrench/wrench": "dev-master"
    }
}
```

#### deps file

If you're using bin/vendors to configure your dependencies, add the following
lines to your `deps` file:

```ini
[wrench]
    git=git://github.com/varspool/Wrench.git
    version=origin/master

[VarspoolWebsocketBundle]
    git=git://github.com/varspool/WebsocketBundle.git
    target=/bundles/Varspool/WebsocketBundle
    version=origin/master
```

Or, fork to your own repository first so you can send in pull requests and
improve upstream :+1:. Once you've done this you can use `bin/vendors` to
obtain the bundles:

```
$ bin/vendors update

[...]
  > Installing/Updating wrench
  > Installing/Updating VarspoolWebsocketBundle
```

##### app/autoload.php

Register the Varspool and Wrench namespaces in your autoloader (not
necessary if you're using Composer with Symfony 2.1):

```php
# app/autoload.php
$loader->registerNamespaces(array(
    // [...]
    'Varspool' => __DIR__.'/../vendor/bundles',
    'Wrench'   => __DIR__.'/../vendor/wrench/lib'
));
```

### app/AppKernel.php

Register the `VarspoolWebsocketBundle`:

```php
# app/AppKernel.php
public function registerBundles()
{
    $bundles = array(
        //...
        new Varspool\WebsocketBundle\VarspoolWebsocketBundle(),
    );
}
```

## Usage

### Server-side starts with a server

Any PHP process can only run a single Websocket server, serving a variable
number of clients. Performance is untested, and once you get into production
you might want to replace the server side. (The Multiplex interfaces descibed
below might help: put a message queue between your PHP application code and a
lightweight Websocket server.)

To start a server, the bundle provides a `websocket:listen` console command,
accessible through `app/console`:

```
Usage:
 websocket:listen [server_name]

Arguments:
 server_name      The server name (from your varspool_websocket configuration) (default: default)
```

The listen command takes a single required argument: the name of the server
configuration to use. Servers are defined in your Symfony2 configuration,
whether YAML, XML or PHP. You must define at least one to get started (we
suggest "default"). Here's what a definition might look like:

```yaml
varspool_websocket:
    servers:
        default: # Server name
            listen: ws://192.168.1.103:8000 # default: ws://localhost:8000

            # Applications this server will allow
            applications:
                - echo
                - multiplex

            # Origin control
            check_origin: true
            allow_origin: # default: just localhost (not useful!)
                - "example.com"
                - "development.localdomain"

            # Other defaults
            max_clients:             30
            max_connections_per_ip:  5
            max_requests_per_minute: 50
```

Once you've configured a server, run the websocket:listen command. When it
runs, the server will start up and serve applications you've
defined in your configuration.

### Applications

The server on its own doesn't do anything until you write an **application**
for it. The server calls methods on your applications once they are registered.

Registering your application is easy. The server looks for services tagged as
`varspool_websocket.application`.  So, to run an application, export a service
with that tag.

A single server daemon can serve one or more applications. So, you'll also have
to include a `key` attribute with your tag. This ends up in your application URL.
For example, if you use this service definition:

```xml
<!-- Application\ChatBundle\Resources\config\services.xml -->
<service id="chat_service" class="Application\ChatBundle\Services\ChatService">
    <tag name="varspool_websocket.application" key="chat" />
</service>
```

And your server is configured to listen on 192.168.1.10:8000, then the URL of
your application will be:

    ws://192.168.1.10:8000/chat

Applications are not registered on servers unless they are specified in the
server configuration. So, to enable the above application on the default
server, you configuration would need to contain:

```
# app/config.yml
varspool_websocket:
    servers:
        default:
            # ...
            applications:
                - chat
```

Here's another example service definition, this time in YAML:

```yaml
services:
    websocket_example:
        class: Application\ExampleBundle\Application\ExampleApplication
        tags:
            - { name: varspool_websocket.application, key: foobar }
```

For a simple example of an application, see `Application\EchoApplication`.

I suggest you make your application classes extend
`Varspool\WebsocketBundle\Application\Application`, but it's optional. A
tiny bit of type checking is done to see if your application would like logging
support, but that's about it. So, you're free to extend whatever class you
like: just implement a compatible interface. (This is the same approach taken
by php-websocket so far: an abstract `WebSocket\Application` class is provided,
but the Server does no typechecking.)

Finally, here's what that the listen command looks like when you run it with a few services
defined:

```
$ app/console websocket:listen default
info: Listening on 192.168.1.103:8080 with ssl off
info: Server created
info: Registering application: test (Application\TestBundle\Application\TestApplication)
info: Registering application: auth (Application\TestBundle\Application\AuthApplication)
info: Registering application: multiplex (Varspool\WebsocketBundle\Application\MultiplexApplication)
info: Registering application: echo (Varspool\WebsocketBundle\Application\EchoApplication)
```

### Client-side

Of course, you'll need a browser that supports websockets.

As for Javascript libraries, they're mostly up to you. But unless you're
already using Coffeescript, you might find the ones shipped along with
php-websocket a pain to install.

### Multiplexing

One thing I would recommend is multiplexing your javascript components'
connections. The SockJS way of doing that is [pretty
elegant](http://www.rabbitmq.com/blog/2012/02/23/how-to-compose-apps-using-websockets/),
and is supported by an application shipped along with this bundle.


#### Client-side multiplexing

This bundle is compatible with the multiplex protocol that the [SockJS
websocket-multiplex front-end
library](https://github.com/sockjs/websocket-multiplex) uses. See
[sockjs/websocket-multiplex](https://github.com/sockjs/websocket-multiplex) for
downloads. They even have a handy CDN:

```html
<script src="http://cdn.sockjs.org/websocket-multiplex-0.1.js"></script>
```

This Javascript library provides a `WebSocketMultiplex` object. You can feed it
any object compatible with a native `WebSocket`. So, to start with you can feed
it a native WebSocket, and later on, when you decide to install a SockJS server
(or one is implemented in PHP) you can feed it a SockJS object. So, like this:

```javascript
var url         = 'ws://example.com:8000/multiplex';

var socket;
if (window.MozWebSocket) {
    socket = new MozWebSocket(url);
} else if (window.WebSocket) {
    socket = new WebSocket(url);
} else {
    throw "No websocket support detected"
}

socket.binaryType = 'blob';

var real_socket = new WebSocket(url);
var multiplexer = new WebSocketMultiplex(real_socket);

var foo  = multiplexer.channel('bar');
// foo.send(), events: open, close, error, message

var logs = mutliplexer.channel('log_server');
// logs.send(), events: open, close, error, message

```

#### Server-side multiplexing

The default configuration for this bundle (in
Varspool/WebsocketBundle/Resources/config/services.xml) defines a server-side
multiplex application, with a key of "multiplex". Make sure this key is listed in
your config, under the allowed applications for your server.


When you're using the multiplex *application*, run by a *server*, your socket
is further abstracted into *channels*, identified by a *topic* string. All the
server-side *listeners* to a channel are notified of each message received from
a client. The listeners can then decide to reply to just the client who sent
the message, or to all clients subscribed to a channel.

* Clients cannot send messages to other clients, unless you specifically relay
  them.
  * Channels provide a handy abstraction to do so: `$channel->send('foo', 'text', false, array('except' => $client))`
* Listeners cannot send messages to other listeners.
  * But you can use whatever you like for that: listeners can be DI'd into the service
    container.

##### The Listener/ConnectionListener interfaces

On the server side, you need only implement `Multiplex\Listener` to be able to
listen to events on a channel:

```php
/**
 * @param Channel $channel   The channel is an object that holds all the active
 *          client connections to a given topic, and all the server-side
 *          subscribers. You can ->send($message) to the channel to broadcast
 *          it to all the subscribed client connections. ->getTopic() identifies
 *          the topic the message was received on.
 *
 * @param string $message    The received message, as a string
 *
 * @param Connection $client The client connection the message was received
 *          from. You can ->send($string) to the client, but it is a raw Websocket
 *          connection, so if you want to send a multiplexed message to a single
 *          client, you'll probably use
 *          `Varspool\WebsocketBundle\Multiplex\Protocol::toString($type, $topic, $payload)`
 *          and the Protocol::TYPE_MESSAGE constant.
 */
public function onMessage(Channel $channel, $message, Connection $client);
```
Then just tag your service with
`varspool_websocket.multiplex_listener` and the topic you want to listen to:

```xml
<service id="example.custom" class="Application\ExampleBundle\Services\CustomService">
    <tag name="varspool_websocket.multiplex_listener" topic="chat" />
</service>
```

All done. Your `onMessage` method will be called with the details of messages
clients sent to the multiplex topic you specified. You might also like to
do something as clients "connect" to (actually, subscribe) and "disconnect"
from (either a real disconnect, or an unsubscribe) your service. To do this,
implement the additional `Multiplex\ConnectionListener` interface as well:

```php
use Varspool\WebsocketBundle\Multiplex\Listener;
use Varspool\WebsocketBundle\Multiplex\ConnectionListener;
use Varspool\WebsocketBundle\Multiplex\Channel;
use WebSocket\Connection;

class GameServer implements Listener, ConnectionListener
{
    public function onMessage(Channel $channel, $message, Connection $client)
    {
        $client->send('Hello, player!');
        $channel->send('Oh, wow, guys...' . $client->getClientId() . ' is here';
    }

    public function onConnect(Channel $channel, Connection $client)
    {
        $client->send('Welcome to the dungeon');
    }

    public function onDisconnect(Channel $channel, Connection $client)
    {
        $channel->send($client->getClientId() . ' is leaving! OH NOES!');
    }
}
```

##### The MultiplexService parent service

For convenience, if you want to implement both of these interfaces, and some
other useful functionality (like getting access to the server or mulitplex
application instances), just extend `Services\MultiplexService`, and export it
in your config with `varspool_websocket.multiplex_service` as its parent.

Here's what that looks like in YAML (with bonus: mulitple channel listener):

```yaml
# config.yml
services:
    example.websocket_auth:
        class:  Application\ExampleBundle\Services\AuthService
        parent: varspool_websocket.multiplex_service
        tags:
            -
                name:  varspool_websocket.multiplex_listener
                topic: auth
            -
                name:  varspool_websocket.multiplex_listener
                topic: login
```
