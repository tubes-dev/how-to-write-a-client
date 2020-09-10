# How To Write Your Own Client
Tubes provides some extra functionality to the websocket connections to allow for peristent client identification as well as direct messaging. This allows a simple websocket connection to act like a fully developed application. However to fully take advantage of these features, you need to develop your client in a way that will support them.

## 🔌 Connecting to a channel.
When a client connects to `tubes.dev`, it will receive two messages. An `identity` message and a `connected` message.

The `identity` message, identifies the client with their own unique ID. All messages are relayed to all clients so if the client doesn't care about their own messages, then this `client_id` can be filtered out. A `jwt` token is supplied for reconnecting so that the client can keep their `client_id`.

The `connected` and corresponding `disconnected` messages, are broadcasted to all connected clients, including the client
that connected. It informs all clients that a client with the client id provided, has connected or disconnected from the channel. It also provides the current count of connected clients.

#### Identity Message
        
```json 
{
  "action": "identity",
  "client_id": "ad29b337-17e1-4a44-926d-1a42b942142e",
  "client_count": 1,
  "jwt": "long JWT string",
}
```

#### Connected Message</p>
```json
{
  "action": "connected",
  "client_id": "ad29b337-17e1-4a44-926d-1a42b942142e",
  "client_count": 1,
}
```

#### Disconnected Message

```json
{
  "action": "disconnected",
  "client_id": "ad29b337-17e1-4a44-926d-1a42b942142e",
  "client_count": 1,
}
```

## 🚦 Reconnecting
When the client receives the `identity` message, a jwt field is included. This is a signed jwt token then contains the client ID so that is can be used to retain client ids during reconnections. Pass the token as a `jwt` parameter on the connection URL while reconnecting.
        
#### Reconnecting
        
```js 
const connectWS = (room, jwt) => {
  let currentJWT = jwt;
  let jwtParam = (jwt ? `?jwt=${jwt}` : '');
  let conn = new WebSocket(`wss://tubes.dev/${room}${jwtParam}`)
  conn.onclose = () => { connectWS(room, currentJWT); }
  conn.onmessage = (msg) => {
    const payload = JSON.parse(msg.data);
    switch(payload.action) {
      case 'identity': currentJWT = payload.jwt;
    }
  }
  return conn
}
connectWS('F3A00E');
```

## 📦 Sending messages in a channel.

All message are signed with the `client_id` of the client that sent the message. This can be used to keep persistent identification of each client connected to a channel.

If this client is sending JSON data, then a `client_id` is added to the object.

If the client is not sending JSON data, then the payload is wrapped in a JSON object and the `client_id` is added to that.

#### Sending JSON data
```js
conn.send(JSON.dump({action: "GREET", payload: "hello"}))
```

```json
{
  "action": "GREET",
  "payload": "hello",
  "client_id": "ad29b337-17e1-4a44-926d-1a42b942142e",
}
```
        
#### Sending non-JSON data
```js
conn.send("coming through the tubes");
```

```json
{
  "payload": "coming through the tubes",
  "client_id": "ad29b337-17e1-4a44-926d-1a42b942142e",
}
```

## 🕵️ Sending messages to a single client.
It is possible to send a message directly to a `client_id` by providing a `to` field in the sent message.

The client that receives the message will have a `private` field, indicating that the message was sent directly to them from the `client_id` in the message.

#### Sending a private message
```js
conn.send(JSON.dump({
  action: "GREET",
  payload: "hello"
  to: "39a4b8c0-8e5f-49d7-b8df-501b25214cbd"
}))
```

#### Client receives:
```json
{
  "action": "GREET",
  "payload": "hello",
  "client_id": "ad29b337-17e1-4a44-926d-1a42b942142e",
  "private": true,
}
```
