---
name: socketbox
description: >
  Use this skill when building WebSocket features with SocketBox in ColdBox/CFML/BoxLang.
  Covers the real SocketBox APIs for core WebSocket handling and STOMP broker usage,
  including correct base classes, method signatures, server-side sends, and client examples.
applyTo: "**/*.{bx,cfc,cfm,bxm}"
---

# SocketBox Skill

## When to Use This Skill

Load this skill when:
- Building WebSocket features in CommandBox or BoxLang MiniServer
- Implementing a `/WebSocket.cfc` listener for connect/close/message lifecycle hooks
- Sending messages to one client or all clients with core WebSocket support
- Using STOMP semantics (destinations, subscriptions, exchanges, auth) when needed

## Installation

```bash
box install socketbox
```

## Configuration

SocketBox is driven by web server settings (CommandBox) and the listener class path.

### CommandBox settings

- `web.websocket.enable` (global) or `sites.mySite.websocket.enable` (per-site)
- `web.websocket.uri` or `sites.mySite.websocket.uri` (default `/ws`)
- `web.websocket.listener` or `sites.mySite.websocket.listener` (default `/WebSocket.cfc`)

### Listener class

Create `/WebSocket.cfc` (or `.bx`) in your web root and extend one of:

- `modules.socketbox.models.WebSocketCore`
- `modules.socketbox.models.WebSocketSTOMP`

## Core WebSocket Mode

Use this for low-level WebSocket support with plain string messages.

### Base class

```js
component extends="modules.socketbox.models.WebSocketCore" {
}
```

### Lifecycle hooks (override these)

- `onConnect( required channel )`
- `onClose( required channel )`
- `onMessage( required message, required channel )`

### Core inherited methods

- `sendMessage( required message, required channel, timeoutMS=0 )`
- `broadcastMessage( required message )`
- `getAllConnections()`

### Core example

```js
component extends="modules.socketbox.models.WebSocketCore" {

    function onConnect( required channel ) {
        sendMessage( "Connected", arguments.channel );
    }

    function onClose( required channel ) {
        // Optional cleanup for disconnected clients
    }

    function onMessage( required message, required channel ) {
        if ( arguments.message EQ "Ping" ) {
            sendMessage( "Pong", arguments.channel );
        }
    }
}
```

## STOMP Broker Mode

Use this when you want destinations, subscriptions, exchange routing, heartbeats,
and optional auth/authz hooks.

### Base class

```js
component extends="modules.socketbox.models.WebSocketSTOMP" {
}
```

### STOMP hooks (override as needed)

- `configure()`
- `authenticate( required string login, required string passcode, string host, required channel )`
- `authorize( required string login, required string exchange, required string destination, required string access, required channel )`
- `onSTOMPConnect( required message, required channel )`
- `onSTOMPDisconnect( required message, required channel )`

### STOMP public server-side methods

- `send( required string destination, required any messageData, struct headers={} )`
- `getSubscriptions()`
- `getExchanges()`
- `getSTOMPConnections()`
- `getConfig()`
- `getConnectionDetails( required channel )`

### STOMP message object methods

- `getCommand()`
- `getBody()`
- `getHeaders()`
- `getHeader( required string key, string defaultValue )`
- `getBodyRaw()`
- `getChannel()`

### STOMP configure example

```js
component extends="modules.socketbox.models.WebSocketSTOMP" {

    /**
     * Called when server starts. If debugMode=true, reloaded every request.
     */
    function configure() {
        return {
            "debugMode"   : false,
            "heartBeatMS" : 10000,
            "exchanges"   : {
                "topic" : {
                    "bindings" : {
                        "myTopic.*" : "destination1"
                    }
                }
            },
            "subscriptions" : {
                "destination1" : ( message ) => {
                    writeDump( var=message.getBody() );
                }
            }
        };
    }
}
```

## Browser/Client Examples

### Basic WebSocket client

```html
<script>
socket = new WebSocket( 'ws://localhost/ws' );

socket.onopen = function() {
    console.log( 'Connected to WebSocket server' );
    socket.send( "Hello from the browser!" );
};

socket.onmessage = function( event ) {
    console.log( 'Message from server:', event.data );
};

socket.onclose = function() {
    console.log( 'Socket is closed.' );
};

socket.onerror = function( err ) {
    console.error( 'Socket encountered error:', err.message );
};
</script>
```

### STOMP client example

```html
<script type="importmap">
{
  "imports": {
    "@stomp/stompjs": "https://ga.jspm.io/npm:@stomp/stompjs@7.0.0/esm6/index.js"
  }
}
</script>
<script type="module">
import { Client } from '@stomp/stompjs';

client = new Client({
  brokerURL: 'ws://localhost/ws',
  reconnectDelay: 5000,
  heartbeatIncoming: 10000,
  heartbeatOutgoing: 10000,
  connectHeaders: {
    login: 'myuser',
    passcode: 'mypass'
  },
  onConnect: () => {
    client.subscribe( 'my-destination', function( message ) {
      console.log( 'Message received', message.body );
    });
  }
});

client.activate();
</script>
```

## Best Practices

- SocketBox models room-like routing via STOMP exchanges, destinations, and subscriptions, not room-named core methods such as joinRoom/leaveRoom.
- Core mode messages are plain strings. Define your own JSON conventions if needed.
- Prefer STOMP mode when you need routing, subscriptions, auth/authz semantics, and heartbeat behavior.
- Use `wss://` in production.
- If clustering is enabled in STOMP config, call inherited `shutdown()` during app shutdown.

## Documentation

- SocketBox README: https://github.com/coldbox-modules/SocketBox#readme
