# WebSocket (lambda-pipe)

lambda-pipe supports AWS API Gateway WebSocket APIs with a simple handler abstraction.

It handles:
- $connect
- $disconnect
- custom route messages

---

# Installation

```bash
npm install @aws-sdk/client-apigatewaymanagementapi
```

---

# Basic Usage

```ts
import { wsHandler } from 'lambda-pipe/websocket'

export const handler = wsHandler({
  connect: async () => {
    console.log('client connected')
  },

  message: async (ctx) => {
    return {
      statusCode: 200,
      body: JSON.stringify({ ok: true }),
    }
  },

  disconnect: async () => {
    console.log('client disconnected')
  },
})
```

---

# Context

```
{
  event: APIGatewayProxyWebsocketEventV2
}
```

---

# Important fields

```
const connectionId =
  context.event.requestContext.connectionId

const routeKey =
  context.event.requestContext.routeKey
```

| Field | Description |
|------|-------------|
| connectionId | unique connection |
| routeKey | $connect / message / disconnect |

---

# Chat Example

```ts
export const handler = wsHandler({
  connect: async (ctx) => {
    console.log('connected')
  },

  message: async (ctx) => {
    const body = JSON.parse(ctx.event.body || '{}')

    return {
      statusCode: 200,
      body: JSON.stringify({
        received: true,
      }),
    }
  },

  disconnect: async () => {
    console.log('disconnected')
  },
})
```

---

# Metal model

```bash
API Gateway WebSocket
   ↓
lambda-pipe wsHandler
   ↓
routeKey dispatcher
   ↓
handler
   ↓
response
```

---

# Notes

- Works only with AWS API Gateway WebSocket
- Stateless execution model
