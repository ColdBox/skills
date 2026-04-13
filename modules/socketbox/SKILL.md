---
name: socketbox
description: >
  Use this skill when building real-time WebSocket applications in ColdBox/BoxLang using socketbox.
  Covers event handler setup, onConnect/onDisconnect/onMessage hooks, broadcasting to rooms,
  client-side JavaScript integration, and production patterns for live notifications and chat.
applyTo: "**/*.{bx,cfc,cfm,bxm}"
---

# Socketbox Skill

## When to Use This Skill

Load this skill when:
- Building real-time features (live notifications, chat, dashboards) with WebSockets
- Handling WebSocket connect, disconnect, and message events in ColdBox
- Broadcasting messages to named rooms or individual clients
- Integrating the WebSocket server with existing ColdBox services

## Installation

```bash
box install socketbox
```

## Configuration

### config/modules/socketbox.cfc

```js
function configure() {
    return {
        // Port for the WebSocket server
        port         : 8080,

        // SSL (set to true in production with a valid certificate)
        ssl          : false,
        sslCert      : "",
        sslKey       : "",

        // Ping/pong heartbeat interval (ms)
        pingInterval : 30000,

        // Max message size (bytes)
        maxPayloadSize: 65536
    }
}
```

## Creating a WebSocket Event Handler

```js
// websockets/ChatHandler.bx
class extends="socketbox.models.BaseEventHandler" {

    property name="log" inject="logbox:logger:{this}";

    // Fires when a client connects
    void function onConnect( socket, event ) {
        log.info( "Client connected: #socket.getId()#" )

        // Send a welcome message to just this client
        socket.send( serializeJSON( {
            type    : "connected",
            message : "Welcome to the chat!"
        } ) )
    }

    // Fires when a client disconnects
    void function onDisconnect( socket, event ) {
        log.info( "Client disconnected: #socket.getId()#" )

        // Notify the room
        broadcast(
            room    = socket.getRoom(),
            message = serializeJSON( { type: "left", socketId: socket.getId() } ),
            exclude = socket.getId()
        )
    }

    // Fires when a message is received from a client
    void function onMessage( socket, message, event ) {
        var payload = deserializeJSON( message )

        switch ( payload.type ) {
            case "join":
                socket.joinRoom( payload.room )
                broadcast(
                    room    = payload.room,
                    message = serializeJSON( { type: "joined", room: payload.room, socketId: socket.getId() } ),
                    exclude = socket.getId()
                )
                break

            case "chat":
                broadcast(
                    room    = socket.getRoom(),
                    message = serializeJSON( {
                        type    : "chat",
                        from    : payload.username,
                        body    : encodeForHTML( payload.body ),
                        sentAt  : dateTimeFormat( now(), "iso8601" )
                    } )
                )
                break

            case "leave":
                socket.leaveRoom()
                break
        }
    }

    // Fires on error
    void function onError( socket, error, event ) {
        log.error( "WebSocket error for socket #socket.getId()#: #error.message#" )
    }
}
```

## Broadcasting

```js
// Send to all clients in a room
broadcast( room = "lobby", message = serializeJSON( notification ) )

// Send to all except one client
broadcast( room = "lobby", message = msg, exclude = socket.getId() )

// Send to a specific socket
socket.send( serializeJSON( { type: "ack", id: messageId } ) )

// Send to all connected clients (no room filter)
broadcastAll( serializeJSON( { type: "announcement", message: "Server restarting in 5 min" } ) )
```

## Client-Side JavaScript

```html
<script>
const ws = new WebSocket( "ws://localhost:8080" )

ws.onopen = () => {
    ws.send( JSON.stringify( { type: "join", room: "lobby" } ) )
}

ws.onmessage = ( event ) => {
    const payload = JSON.parse( event.data )

    if ( payload.type === "chat" ) {
        appendChatMessage( payload.from, payload.body )
    }
}

ws.onclose = () => console.log( "Disconnected" )

function sendChat( body ) {
    ws.send( JSON.stringify( { type: "chat", username: currentUser, body: body } ) )
}
</script>
```

## Production Patterns

### Server-Push Notifications (from a ColdBox Handler)

```js
// ColdBox handler pushes a notification via socketbox
property name="socketbox" inject="SocketBox@socketbox";

function approve( event, rc, prc ) {
    approvalService.approve( rc.id )

    // Push real-time update to the relevant room
    socketbox.broadcast(
        room    = "order_#rc.orderId#",
        message = serializeJSON( {
            type    : "order_approved",
            orderId : rc.orderId,
            ts      : dateTimeFormat( now(), "iso8601" )
        } )
    )

    event.renderData( type = "json", data = { success: true } )
}
```

### Authentication on Connect

```js
void function onConnect( socket, event ) {
    var token  = socket.getHeader( "Authorization" ) ?: ""
    var userId = jwtService.parseToken( token )

    if ( isNull( userId ) ) {
        socket.send( serializeJSON( { type: "error", message: "Unauthorized" } ) )
        socket.close( 4001, "Unauthorized" )
        return
    }

    socket.setAttribute( "userId", userId )
    socket.joinRoom( "user_#userId#" )
}
```

## Best Practices

- **Validate and sanitize** all incoming message payloads — never trust client-supplied data
- **Encode user-generated text** (`encodeForHTML`) before broadcasting to chat rooms
- **Authenticate on connect** using a short-lived token (JWT or session token) — not per message
- **Use rooms** to scope broadcasts — avoid broadcasting to all clients unless truly needed
- **Cap message size** with `maxPayloadSize` to prevent memory exhaustion attacks
- **Use WSS (TLS)** in production — plain WebSocket connections expose data in transit
- **Handle `onError` and `onDisconnect`** — clean up rooms and notify other participants

## Documentation

- socketbox: https://github.com/coldbox-modules/socketbox
