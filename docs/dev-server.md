# Dev Server (lambda-pipe)

Local development server that simulates AWS Lambda execution using a Fastify-based runtime.

It converts HTTP requests into Lambda-like events and executes routes with full middleware + plugin lifecycle.

---

# Features

- File-based routing (`src/routes`)
- Lambda-like request context
- Fastify-based dev runtime
- Cookie support
- JSON-safe response serialization
- Dynamic route loading
- Optional hot reload (tsx / nodemon)

---

# Installation

```bash
npm install fastify @fastify/cookie
```

---

# Quick Start

```ts
import { bootstrap } from 'lambda-pipe/cli'

await bootstrap({
  port: 3000,
  routesDir: 'src/routes',
})

```

```bash
npm run dev
```

---

# Project Structure

```bash
src/
  routes/
    health.ts
    user.ts
```

---

# Health Route

```ts
import { defineRoute, ok } from 'lambda-pipe'

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

# Full User Route Example

```ts
import {
  defineRoute,
  ok,
  HttpError,
  ValidationError,
} from 'lambda-pipe'

import type { Middleware, Plugin, Validator } from 'lambda-pipe'

type User = {
  id: string
  teamId: string
}

type PluginContext = {
  log: {
    info: (...args: any[]) => void
    error: (...args: any[]) => void
  }

  auth: {
    can: (scope: string) => boolean
  }

  db: {
    findUserById: (id: string) => Promise<any>
  }
}

const paramsValidator: Validator<{ id: string }> = async (input) => {
  const params = input as any
  if (!params.id) throw new ValidationError('missing id')
  return { id: params.id }
}

const queryValidator: Validator<{ expand?: string }> = async (input) => {
  const query = input as any
  if (query.expand && typeof query.expand !== 'string') {
    throw new ValidationError('expand must be string')
  }
  return { expand: query.expand }
}

const loggerPlugin: Plugin = {
  name: 'logger',
  extend: () => ({
    log: {
      info: console.log,
      error: console.error,
    },
  }),
}

const authPlugin: Plugin = {
  name: 'auth',
  extend: () => ({
    auth: {
      can: (scope: string) => scope === 'read:user',
    },
  }),
}

const dbPlugin: Plugin = {
  name: 'db',
  extend: () => ({
    db: {
      findUserById: async (id: string) => ({
        id,
        username: 'john_doe',
        createdAt: new Date().toISOString(),
      }),
    },
  }),
}

const requestMiddleware: Middleware<any, User, PluginContext> = {
  onRequest: async (context, next) => {
    context.state.startedAt = Date.now()

    context.log.info('request start', {
      requestId: context.req.requestId,
      ip: context.req.ip,
    })

    if (!context.req.user) {
      throw new HttpError(401, 'Unauthorized')
    }

    if (!context.auth.can('read:user')) {
      throw new HttpError(403, 'Forbidden')
    }

    await next()
  },

  onResponse: async (context) => {
    const ms = Date.now() - context.state.startedAt
    context.log.info('response done', { duration: ms })
  },

  onError: async (err, context) => {
    context.log.error('request failed', {
      err,
      requestId: context.req.requestId,
    })
  },
}

export default defineRoute<
  {
    params: { id: string }
    query: { expand?: string }
  },
  User,
  PluginContext
>({
  method: 'GET',
  path: '/user/:id',

  plugins: [loggerPlugin, authPlugin, dbPlugin],
  middleware: [requestMiddleware],

  validate: {
    params: paramsValidator,
    query: queryValidator,
  },

  authenticate: async (token) => {
    if (token !== 'valid-token') return null

    return {
      user: {
        id: 'u1',
        teamId: 'team-1',
      },
      scope: ['read:user'],
      metadata: {
        role: 'admin',
      },
    }
  },

  handler: async (context) => {
    const { id } = context.params
    const { expand } = context.query

    const cacheKey = `user:${id}`
    const cached = context.exec.cache.get(cacheKey)

    if (cached) {
      return ok({
        source: 'cache',
        data: cached,
      })
    }

    const user = await context.db.findUserById(id)

    const result = {
      id: user.id,
      username: user.username,
      teamId: context.req.user?.teamId,
      expand,
      createdAt: user.createdAt,
    }

    context.exec.cache.set(cacheKey, result, 10000)

    return ok({
      source: 'database',
      data: result,
    })
  },
})
```

---

# Execution Flow

```
HTTP Request
  ↓
Fastify server
  ↓
Route loader
  ↓
Lambda-like context
  ↓
Middleware + Auth
  ↓
Validation
  ↓
Plugins injection
  ↓
Handler
  ↓
Response
```