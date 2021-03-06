[![Join the chat at https://t.me/joinchat/ABFVWBE0AhkyyhREoaboXQ](https://img.shields.io/badge/Telegram-Group-blue.svg)](https://t.me/joinchat/ABFVWBE0AhkyyhREoaboXQ)
[![Build Status](https://travis-ci.org/centrifugal/centrifuge.svg?branch=master)](https://travis-ci.org/centrifugal/centrifuge)
[![Coverage Status](https://coveralls.io/repos/github/centrifugal/centrifuge/badge.svg?branch=master)](https://coveralls.io/github/centrifugal/centrifuge?branch=master)
[![GoDoc](https://godoc.org/github.com/centrifugal/centrifuge?status.svg)](https://godoc.org/github.com/centrifugal/centrifuge)

**This library has no v1 release yet so API can be changed. Use with strict versioning.**

Centrifuge library is a real-time core of [Centrifugo](https://github.com/centrifugal/centrifugo) server. It's also supposed to be a general purpose real-time messaging library for Go programming language. The library is based on a strict client-server protocol based on Protobuf schema and solves several problems developer may come across when building complex real-time applications – like scalability (millions of connections), proper connection management, fast reconnect with message recovery, fallback option.

Library highlights:

* Fast and optimized for low-latency communication with thousands of client connections. See [benchmark](https://centrifugal.github.io/centrifugo/misc/benchmark/)
* WebSocket with JSON or binary Protobuf protocol
* SockJS polyfill library support for browsers where WebSocket not available (JSON only)
* Built-in horizontal scalability with Redis PUB/SUB, Redis sharding, Sentinel for HA
* Possibility to register custom PUB/SUB broker, history and presence storage implementations
* Native authentication over HTTP middleware or JWT-based
* Bidirectional asynchronous message communication and RPC calls
* Channel (room) concept to broadcast message to all channel subscribers
* Presence information for channels (show all active clients in channel)
* History information for channels (last messages published into channel)
* Join/leave events for channels (aka client goes online/offline)
* Message recovery mechanism for channels to survive short network disconnects or node restart
* Prometheus instrumentation
* Client libraries for main application environments (see below)

Client libraries:

* [centrifuge-js](https://github.com/centrifugal/centrifuge-js) – for browser, NodeJS and React Native
* [centrifuge-go](https://github.com/centrifugal/centrifuge-go) - for Go language
* [centrifuge-mobile](https://github.com/centrifugal/centrifuge-mobile) - for iOS and Android using `centrifuge-go` as basis and `gomobile` project to create bindings
* [centrifuge-dart](https://github.com/centrifugal/centrifuge-dart) - for Dart and Flutter
* [centrifuge-swift](https://github.com/centrifugal/centrifuge-swift) – for native iOS development
* [centrifuge-java](https://github.com/centrifugal/centrifuge-java) – for native Android development and general Java

See [Godoc](https://godoc.org/github.com/centrifugal/centrifuge) and [examples](https://github.com/centrifugal/centrifuge/tree/master/_examples). You can also consider [Centrifugo server documentation](https://centrifugal.github.io/centrifugo/) as extra doc for this package (because it's built on top of Centrifuge library).

### Installation

To install use:

```bash
go get github.com/centrifugal/centrifuge
```

`go mod` is a recommended way of adding this library to your project dependencies.

### Quick example

Let's take a look on how to build the simplest real-time chat ever with Centrifuge library. Clients will be able to connect to server over Websocket, send a message into channel and this message will be instantly delivered to all active channel subscribers. On server side we will accept all connections and will work as simple PUB/SUB proxy without worrying too much about permissions. In this example we will use Centrifuge Javascript client on a frontend.

Create file `main.go` with the following code:

```go
package main

import (
	"context"
	"log"
	"net/http"
	"os"
	"os/signal"
	"syscall"
	"time"

	// Import this library.
	"github.com/centrifugal/centrifuge"
)

func handleLog(e centrifuge.LogEntry) {
	log.Printf("%s: %v", e.Message, e.Fields)
}

// Wait until program interrupted. When interrupted gracefully shutdown Node.
func waitExitSignal(n *centrifuge.Node) {
	sigCh := make(chan os.Signal, 1)
	done := make(chan bool, 1)
	signal.Notify(sigCh, syscall.SIGINT, syscall.SIGTERM)
	go func() {
		<-sigCh
		ctx, cancel := context.WithTimeout(context.Background(), 3*time.Second)
		defer cancel()
		_ = n.Shutdown(ctx)
		done <- true
	}()
	<-done
}

func main() {
	// We use default config here as starting point. Default config contains
	// reasonable values for available options.
	cfg := centrifuge.DefaultConfig
	// In this example we want client to do all possible actions with server
	// without any authentication and authorization. Insecure flag DISABLES
	// many security related checks in library. This is only to make example
	// short. In real app you most probably want authenticate and authorize
	// access to server. See godoc and examples in repo for more details.
	cfg.ClientInsecure = true
	// By default clients can not publish messages into channels. Setting this
	// option to true we allow them to publish.
	cfg.Publish = true

	// Centrifuge library exposes logs with different log level. In your app
	// you can set special function to handle these log entries in a way you want.
	cfg.LogLevel = centrifuge.LogLevelDebug
	cfg.LogHandler = handleLog

	// Node is the core object in Centrifuge library responsible for many useful
	// things. Here we initialize new Node instance and pass config to it.
	node, _ := centrifuge.New(cfg)

	// ClientConnected node event handler is a point where you generally create a 
	// binding between Centrifuge and your app business logic. Callback function you 
	// pass here will be called every time new connection established with server. 
	// Inside this callback function you can set various event handlers for connection.
	node.On().ClientConnected(func(ctx context.Context, client *centrifuge.Client) {
		// Set Subscribe Handler to react on every channel subscribtion attempt
		// initiated by client. Here you can theoretically return an error or
		// disconnect client from server if needed. But now we just accept
		// all subscriptions.
		client.On().Subscribe(func(e centrifuge.SubscribeEvent) centrifuge.SubscribeReply {
			log.Printf("client subscribes on channel %s", e.Channel)
			return centrifuge.SubscribeReply{}
		})

		// Set Publish Handler to react on every channel Publication sent by client.
		// Inside this method you can validate client permissions to publish into
		// channel. But in our simple chat app we allow everyone to publish into
		// any channel.
		client.On().Publish(func(e centrifuge.PublishEvent) centrifuge.PublishReply {
			log.Printf("client publishes into channel %s: %s", e.Channel, string(e.Data))
			return centrifuge.PublishReply{}
		})

		// Set Disconnect Handler to react on client disconnect events.
		client.On().Disconnect(func(e centrifuge.DisconnectEvent) centrifuge.DisconnectReply {
			log.Printf("client disconnected")
			return centrifuge.DisconnectReply{}
		})

		// In our example transport will always be Websocket but it can also be SockJS.
		transportName := client.Transport().Name()
		// In our example clients connect with JSON protocol but it can also be Protobuf.
		transportEncoding := client.Transport().Encoding()

		log.Printf("client connected via %s (%s)", transportName, transportEncoding)
	})

	// Run node.
	if err := node.Run(); err != nil {
		panic(err)
	}

	// Configure http routes.

	// The first route is for handling Websocket connections.
	http.Handle("/connection/websocket", centrifuge.NewWebsocketHandler(node, centrifuge.WebsocketConfig{}))

	// The second route is for serving index.html file.
	http.Handle("/", http.FileServer(http.Dir("./")))

	// Start HTTP server.
	go func() {
		if err := http.ListenAndServe(":8000", nil); err != nil {
			panic(err)
		}
	}()

	// Run until interrupted.
	waitExitSignal(node)
}
```

Also create file `index.html` near `main.go` with content:

```html
<!DOCTYPE html>
<html lang="en">
    <head>
        <meta charset="utf-8">
        <script type="text/javascript" src="https://rawgit.com/centrifugal/centrifuge-js/master/dist/centrifuge.min.js"></script>
        <title>Centrifuge library chat example</title>
    </head>
    <body>
        <input type="text" id="input" />
        <script type="text/javascript">
            // Create Centrifuge object with Websocket endpoint address set in main.go
            const centrifuge = new Centrifuge('ws://localhost:8000/connection/websocket');
            function drawText(text) {
                const div = document.createElement('div');
                div.innerHTML = text + '<br>';
                document.body.appendChild(div);
            }
            centrifuge.on('connect', function(ctx){
                drawText('Connected over ' + ctx.transport);
            });
            centrifuge.on('disconnect', function(ctx){
                drawText('Disconnected: ' + ctx.reason);
            });
            const sub = centrifuge.subscribe("chat", function(ctx) {
                drawText(JSON.stringify(ctx.data));
            })
            const input = document.getElementById("input");
            input.addEventListener('keyup', function(e) {
                if (e.keyCode === 13) {
                    sub.publish(this.value);
                    input.value = '';
                }
            });
            // After setting event handlers – initiate actual connection with server.
            centrifuge.connect();
        </script>
    </body>
</html>
```

Then run as usual:

```bash
go run main.go
```

Open several browser tabs with http://localhost:8000 and see chat in action.

This example is only the top of an iceberg. Though it should give you an insight on library API. 

Keep in mind that Centrifuge library is not a framework to build chat apps. It's a general purpose real-time transport for your messages with some helpful primitives. You can build many kinds of real-time apps on top of this library including chats but depending on application you may need to write business logic yourself.
