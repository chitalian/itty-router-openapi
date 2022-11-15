# itty-router-openapi

This library provides an easy and compact OpenAPI 3 schema generator and validator
for [Cloudflare Workers](https://developers.cloudflare.com/workers/)

`itty-router-openapi` as the name says is built on top of the
awesome [itty-router](https://github.com/kwhitley/itty-router),
while improving some core features such as adding class based endpoints.

This library was designed to provide a simple and iterative path for old `itty-router` applications to migrate to this
new
Router.

This package is still is **development** functions and interfaces documented here are very likelly to never change, but
internal functions
and procedures will probably changes during the initial weeks of publishing.

## Features

- [x] Drop-in replacement for existing itty-router applications
- [x] OpenAPI 3 schema generator
- [x] Fully written in typescript
- [x] Class based endpoints
- [x] Query parameters validator
- [x] Path parameters validator
- [ ] Body request validator

## Installation

```
npm i @cloudflare/itty-router-openapi --save
```

## FAQ

Q. Is this package production ready?

A. Yes! This package was created during the [Cloudflare Radar 2.0](https://radar.cloudflare.com/) development and is
currently
use by the Radar website to serve not only the user faced website but also the public API.

---

Q. When will this package reach stable maturity?

A. While `OpenAPIRouter` function and the `Route` class are not likely to change, internal procedures will be updated
during this initial
beta release.

## Basic Usage

Creating a new OpenAPI route is simple, just create a new class that extends the base `Route` and fill your schema
parameters
and add your code in the handle function.

In the example bellow, the `ToDoList` route will have an Integer parameter called `page` that will be validated before
calling the
`handle()` function. Then inside the `handle()` function the page number will be available in the data object passed in
the argument.

If you try to send a value that is not a Integer in this field a `ValidationError` will be raised that the Route will
internally convert
into a readable http 400 error.

```ts
import { OpenAPIRoute, Query, Int, Str } from '@cloudflare/itty-router-openapi'

export class ToDoList extends OpenAPIRoute {
  static schema = {
    tags: ['ToDo'],
    summary: 'Create a new Todo',
    parameters: {
      page: Query(Int, {
        description: 'Page number',
        default: 1,
        required: false,
      }),
    },
    responses: {
      '200': {
        schema: {
          currentPage: new Int(),
          nextPage: new Int(),
          results: [new Str({ example: 'lorem' })],
        },
      },
    },
  }

  async handle(request: Request, data: Record<string, any>) {
    const { page } = data

    return {
      currentPage: page,
      nextPage: page + 1,
      results: ['lorem', 'ipsum'],
    }
  }
}
```

Then, ideally in a different file, you can register the routes normally

```ts
import { OpenAPIRouter } from '@cloudflare/itty-router-openapi'

const router = OpenAPIRouter()
router.get('/todos', ToDoList)

// 404 for everything else
router.all('*', () => new Response('Not Found.', { status: 404 }))

addEventListener('fetch', (event) => event.respondWith(router.handle(event.request)))
```

Now, when running `wrangler dev` and going to the `/docs` or `/redocs` path you are greeted with an openapi ui that you
can call
your endpoints from the comfort of your browser.

## Migrating from existing `itty-router` applications

After installing just replace the old `Router` function with the new `OpenAPIRouter` function.

```ts
// Old router
//import { Router } from 'itty-router'
//const router = Router()

// New router
import { OpenAPIRouter } from '@cloudflare/itty-router-openapi'

const router = OpenAPIRouter()

// Old routes remain the same
router.get('/todos', () => new Response('Todos Index!'))
router.get('/todos/:id', ({ params }) => new Response(`Todo #${params.id}`))

// ...
```

Now, when running the application and going to the `/docs` path, you will already see all your endpoints listed with the
query
parameters parsed ready to be invoked.

All of this while changing just one line in your existing code base!

## Schema Types

Schema Types can be used both in parameters and responses.

Available Schema Types:

| Name       |                               Arguments                               |
| ---------- | :-------------------------------------------------------------------: |
| `Num`      |     `description` `example` `default` `enum` `enumCaseSensitive`      |
| `Int`      |     `description` `example` `default` `enum` `enumCaseSensitive`      |
| `Str`      | `description` `example` `default` `enum` `enumCaseSensitive` `format` |
| `DateTime` |     `description` `example` `default` `enum` `enumCaseSensitive`      |
| `DateOnly` |     `description` `example` `default` `enum` `enumCaseSensitive`      |
| `Bool`     |     `description` `example` `default` `enum` `enumCaseSensitive`      |

In order to make use of the `enum` argument you should pass your Enum values to the `Enumeration` class, has shown bellow.

Example parameters:

```ts
parameters = {
  page: Query(Number, {
    description: 'Page number',
    default: 1,
    required: false,
  }),
  search: Query(
    new Str({
      description: 'Search query',
      example: 'funny people',
    }),
    { required: false }
  ),
}
```

Example responses:

```ts
responses = {
  '200': {
    schema: {
      result: {
        meta: {
          confidenceInfo: schemaDateRange,
          dateRange: schemaDateRange,
          aggInterval: schemaAggInterval,
          lastUpdated: new DateTime(),
        },
        series: {
          timestamps: [new DateTime()],
          values: [new Str({ example: 0.56 })],
        },
      },
    },
  },
}
```

Example Enumeration:

```ts
import { Enumeration } from '@cloudflare/itty-router-openapi'

const formatsEnum = new Enumeration({
  json: 'json',
  csv: 'csv',
})

parameters = {
  format: Query(formatsEnum, {
    description: 'Format the response should be returned',
    default: 'json',
    required: false,
  }),
}
```

## Parameters

Currently there are support for both `Query` and `Path` parameters

This is where you will use the Schema Types explained above

Example Path Parameter

```ts
import { OpenAPIRoute, Path, Int, Str } from '@cloudflare/itty-router-openapi'

export class ToDoFetch extends OpenAPIRoute {
  static schema = {
    tags: ['ToDo'],
    summary: 'Fetch a ToDo',
    parameters: {
      todoId: Path(Int, {
        description: 'ToDo ID',
      }),
    },
  }

  async handle(request: Request, data: Record<string, any>) {
    const { todoId } = data
    // ...
  }
}

router.get('/todos/:todoId', ToDoFetch)
```

Example Query Parameter

```ts
import { OpenAPIRoute, Query, Int, Str } from '@cloudflare/itty-router-openapi'

export class ToDoList extends OpenAPIRoute {
  static schema = {
    tags: ['ToDo'],
    summary: 'Create a new Todo',
    parameters: {
      page: Query(Int, {
        description: 'Page number',
        default: 1,
        required: false,
      }),
    },
  }

  async handle(request: Request, data: Record<string, any>) {
    const { page } = data
    // ...
  }
}

router.get('/todos', ToDoList)
```

## Advanced Usage

### 1. Using Typescript types

If you are planning on using this lib with typescript, then declaring schemas is even easier than javascript, because
instead of importing the parameter types, you can just use the native typescript data types `String`, `Number`,
or `Boolean`.

```ts
export class ToDoList extends OpenAPIRoute {
  static schema = {
    tags: ['ToDo'],
    summary: 'Create a new Todo',
    parameters: {
      page: Query(Number, {
        description: 'Page number',
        default: 1,
        required: false,
      }),
    },
    responses: {
      '200': {
        schema: {
          currentPage: Number,
          nextPage: Number,
          results: [String],
        },
      },
    },
  }
  // ...
}
```

### 2. Build your own Schema Type

All schema types extend from the `BaseParameter` or other type and build on top of that.
To build your own type just pick an already available type, like `Str` or extend from the base class.

```ts
export class Num extends BaseParameter {
  type = 'number'

  validate(value: any): any {
    value = super.validate(value)

    value = Number.parseFloat(value)

    if (isNaN(value)) {
      throw new ValidationError('is not a valid number')
    }

    return value
  }
}

export class DateOnly extends Str {
  type = 'string'
  protected declare params: StringParameterType

  constructor(params?: StringParameterType) {
    super({
      example: '2022-09-15',
      ...params,
      format: 'date',
    })
  }
}
```

### 3. Core openapi schema customizations

Besides adding a schema to your endpoints, its also recomended to customize your schema.

This can be done by passing the schema argument when creating your router, and all [OpenAPI Fixed Fields](https://swagger.io/specification/#schema) are available.

The example bellow will change the schema title, and add a Bearer token authentication to all endpoints

```ts
const router = OpenAPIRouter({
  schema: {
    info: {
      title: 'Radar Worker API',
      version: '1.0',
    },
    components: {
      securitySchemes: {
        bearerAuth: {
          type: 'http',
          scheme: 'bearer',
        },
      },
    },
    security: [
      {
        bearerAuth: [],
      },
    ],
  },
})
```

### 4. Hiding routes in the OpenAPI schema

Hiding routes can be archived by registering your endpoints in the original `itty-router`, as shown here

```ts
import { OpenAPIRouter } from '@cloudflare/itty-router-openapi'

const router = OpenAPIRouter()

router.original.get('/todos/:id', ({ params }) => new Response(`Todo #${params.id}`))
```

This endpoint will still be accessible, but will not be shown in the schema

## Feedback and contributions

Currently this package is maintained by the [Cloudflare Radar Team](https://radar.cloudflare.com/) and features are prioritized
based on future Radar needs

Nonetheless you can still open pull requests or issues in this repository and they will get addressed

You can also talk to us in the [Cloudflare Community](https://community.cloudflare.com/) or the [Radar Discord Channel](https://discord.com/channels/595317990191398933/1035553707116478495)