# Universal Route Core with AWS + Fastify Adapters

A lightweight architecture pattern that allows a single route definition to run seamlessly across multiple runtimes (AWS Lambda and Fastify) using adapter-based abstraction.

---

## Concept Overview

This architecture separates **core business logic** from **platform-specific implementations** using adapters.

* The **core route** is completely platform-agnostic
* Each runtime (AWS Lambda, Fastify, etc.) only provides an adapter layer
* The same handler logic runs everywhere without modification

---

## 1. Define a Platform-Agnostic Route (Core)

```ts
// src/routes/user.ts

import {
  defineRoute,
  ok,
} from 'lambda-pipe'

export default defineRoute({
  method: 'GET',
  path: '/user/:id',

  handler: async (ctx) => {
    return ok({
      id: ctx.params.id,
      source: ctx.event ? 'lambda' : 'server',
    })
  },
})
```

### Key Idea

* No dependency on AWS, Fastify, or HTTP frameworks
* Only uses a unified `ctx` object
* Business logic is fully isolated

---

## 2. AWS Lambda Adapter

```ts
// src/runtime/aws/handler.ts

import route from '../../routes/user'

import {
  awsAdapter,
  executeRoute,
} from 'lambda-pipe/runtime'

export const handler = async (event: any, context: any) => {
  const req = awsAdapter.normalizeRequest(event)

  const response = await executeRoute(() =>
    route.handler({
      event,
      awsContext: context,
      ...req,
    })
  )

  return awsAdapter.sendResponse(response)
}
```

### Responsibilities

* Convert AWS event → normalized request
* Inject AWS context into `ctx`
* Transform response back to API Gateway format

---

## 3. Fastify Adapter

```ts
// src/runtime/fastify/handler.ts

import Fastify from 'fastify'
import route from '../../routes/user'

import {
  fastifyAdapter,
  executeRoute,
} from 'lambda-pipe/runtime'


const app = Fastify()

app.all('/user/:id', async (req, reply) => {
  const normalized = fastifyAdapter.normalizeRequest(req)

  const response = await executeRoute(() =>
    route.handler({
      event: req,
      awsContext: null,
      ...normalized,
    })
  )

  return fastifyAdapter.sendResponse(response, reply)
})

export default app
```

### Responsibilities

* Convert Fastify request → normalized format
* Map response back to Fastify reply object

---

## 4. Core Principle

🔥 The route logic never changes

| Layer           | Responsibility                |
| --------------- | ----------------------------- |
| Core Route      | Business logic only           |
| AWS Adapter     | Event transformation          |
| Fastify Adapter | HTTP request/response mapping |

---

## 5. Architecture Flow

```
           ┌──────────────┐
           │  Core Route   │
           │ defineRoute() │
           └──────┬───────┘
                  │
     ┌────────────┼────────────┐
     │                         │
┌────▼─────┐           ┌───────▼─────┐
│ AWS Adap │           │ Fastify Adap│
│ ter      │           │ ter         │
└────┬─────┘           └───────┬─────┘
     │                         │
 Lambda Event           HTTP Request
```

---

## 6. Why This Architecture Works

### ✔ Separation of Concerns

* Business logic is isolated from transport layer

### ✔ Reusability

* One route runs everywhere

### ✔ Testability

* Core logic can be tested without AWS or HTTP server

### ✔ Flexibility

* Easily add new runtimes (Express, Cloudflare Workers, etc.)

---

## 7. Summary

Adapters are responsible for:

* Converting request formats
* Converting response formats
* Keeping core logic clean and platform-independent

> One core → multiple runtimes

---

## License

MIT (or your project license)
