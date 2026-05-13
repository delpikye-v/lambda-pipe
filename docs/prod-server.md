# Production Runtime production.ts

Production-ready Fastify runtime for LambdaPipe.

Designed for:

* bundled deployments
* lazy-loaded route manifests
* serverless adapters
* container deployments
* runtime middleware pipelines
* runtime plugin lifecycle hooks

Unlike the dev runtime, production mode does not scan the filesystem dynamically.

Routes are registered through a static manifest for deterministic builds and deployment-safe startup.

---

# Features

* Fastify production runtime
* Static route manifest system
* Lazy-loaded route modules
* Runtime middleware lifecycle
* Runtime plugin hooks
* Shared execution context
* Cookie support
* Response serialization
* Route module cache
* Error handling pipeline
* Deployment-safe routing
* Tree-shake friendly architecture

---

# Installation

```bash
npm install fastify @fastify/cookie
```

---

# Quick Start

```ts
import { bootstrap } from 'lambda-pipe/server'

import routes from './routes.manifest'

await bootstrap({
  port: 3000,

  routes,
})
```

---

# Route Manifest

```ts
// routes.manifest.ts

import type {
  RouteLoader,
} from 'lambda-pipe/server'

const routes: RouteLoader[] = [
  () => import('./routes/health'),

  () => import('./routes/user'),
]

export default routes
```

---

# Why Manifest Loaders

Each manifest item is only responsible for lazy-importing a route module.

Route metadata is defined directly inside `defineRoute()`.

This avoids:

* duplicated route metadata
* route mismatch bugs
* startup filesystem scanning
* runtime path resolution issues

---

# Project Structure

```bash
src/
  routes/
    health.ts
    user.ts

  routes.manifest.ts
```

---

# Health Route

```ts
import {
  defineRoute,
  ok,
} from 'lambda-pipe'

export default defineRoute({
  method: 'GET',

  path: '/health',

  handler: async () => {
    return ok({
      status: 'ok',

      uptime: process.uptime(),
    })
  },
})
```

---

# Full Runtime Route Example

```ts
import {
  defineRoute,
  ok,
  HttpError,
} from 'lambda-pipe'

export default defineRoute({
  method: 'GET',

  path: '/user/:id',

  middlewares: [
    async (context) => {
      if (!context.headers.authorization) {
        throw new HttpError(
          401,
          'Unauthorized',
        )
      }
    },
  ],

  handler: async (context) => {
    const userId =
      context.pathParameters.id

    return ok({
      id: userId,

      source: 'runtime',

      requestId:
        context.requestId,
    })
  },
})
```

---

# Runtime Plugins

```ts
import type {
  RuntimePlugin,
} from 'lambda-pipe/server'

export const loggerPlugin: RuntimePlugin = {
  name: 'logger',

  onInit: async (app) => {
    app.log.info(
      'logger initialized',
    )
  },

  onRequest: async (context) => {
    context.request.log.info({
      method: context.method,

      path: context.rawPath,
    })
  },

  onResponse: async (
    context,
    response,
  ) => {
    context.request.log.info({
      statusCode:
        response.statusCode,
    })
  },

  onError: async (
    error,
    context,
  ) => {
    context.request.log.error(
      error,
    )
  },
}
```

---

# Using Plugins

```ts
import { bootstrap } from 'lambda-pipe/server'

import routes from './routes.manifest'

import { loggerPlugin } from './plugins/logger'

await bootstrap({
  routes,

  plugins: [loggerPlugin],
})
```

---

# Global Middleware

```ts
await bootstrap({
  port: 3000,

  routes,

  logger: true,

  globalMiddlewares: [
    async (context) => {
      context.request.log.info({
        path: context.rawPath,
      })
    },
  ],
})
```

---

# Runtime Lifecycle

```text
HTTP Request
  ↓
Fastify Runtime
  ↓
Route Manifest Match
  ↓
Lazy Route Import
  ↓
Create Runtime Context
  ↓
Plugin onRequest Hooks
  ↓
Global Middlewares
  ↓
Route Middlewares
  ↓
Route Handler
  ↓
Plugin onResponse Hooks
  ↓
Serialize Response
  ↓
HTTP Response
```

---

# Error Lifecycle

```text
Handler Error
  ↓
Plugin onError Hooks
  ↓
Runtime Error Handler
  ↓
Serialized Error Response
```

---

# Response Shape

```ts
return {
  statusCode: 200,

  headers: {
    'x-runtime': 'lambda-pipe',
  },

  body: {
    ok: true,
  },
}
```

---

# Module Cache

Production runtime caches loaded route modules internally using a WeakMap-based loader cache.

Benefits:

* avoids duplicate imports
* keeps lazy-loading efficient
* reduces runtime overhead
* keeps route identity stable

---

# Deployment Targets

Production runtime works well with:

* AWS Lambda
* ECS / Fargate
* Docker containers
* Bun runtime
* Node.js servers
* Edge-compatible adapters
* Kubernetes
* Railway / Render / Fly.io

---

# Why Production Runtime Avoids FS Scanning

Filesystem scanning is avoided in production because:

* faster startup
* deployment-safe bundling
* smaller runtime surface
* deterministic route registration
* serverless compatibility
* better tree-shaking support
* fewer runtime filesystem assumptions

---

# Example Production Bootstrap

```ts
import { bootstrap } from 'lambda-pipe/server'

import routes from './routes.manifest'

await bootstrap({
  port: 8080,

  host: '0.0.0.0',

  routes,

  logger: true,
})
```

---

# Example Startup Logs

```text
⚡ GET /health
⚡ GET /user/:id

🚀 LambdaPipe Runtime
🌐 http://0.0.0.0:3000
📦 routes: 2
```
