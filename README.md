# @adaptivo/*
Adaptivo framework.

A Lowest Common Denominator approach to client-server interactions.

Used for writing portable JavaScript applications with different server and request backends.

## Packages

### @adaptivo/feserver
A client-side interface to request data and changes from the server.

### @adaptivo/router
A server-side interface to route and respond to requests.
Uses URL path compatible keys. No path parameters allowed.

## Design Constraints
The `adaptivo` framework does not attempt to make full use of RESTful API design.
Its main purpose is to provide an adapter between frameworks
like Meteor, Express, Next, and simulated environments.

This is so that applications are decoupled from the underlying server implementation.

As a result, it currently is concerned with just two kinds of requests:

- __quasi-idempotent__ requests (e.g. GET), where side-effects on the server-side are not important, and we just need to fetch data efficiently.
- __mutating__ requests (e.g. POST), where side-effects on the server-side matter, and we want to change the state of our data.

And the main "langauge" of the data, while dependant on adapters, must remain consistent.
For that reason, plain serializable JavaScript objects (like JSON) are used.

Other frameworks should be used for serving HTML, XML, and other formats.

## Usage example

Using `@adaptivo/clientserver` on the client, you would fetch data like so:

```javascript
import { createServer } from "@adaptivo/feserver";
// OR import { server } from "@adaptivo/feserver";
import { connect, disconnect } from "@adaptivo/meteor/feserver";
import { Doc } from "./schema";

const server = createServer()

// Currently only one runtime at a time can be used to get data for the server.
// In the future, multiple suppliers could be used collaboratively, if a use case comes up.
connect(server);

const response = await server.get("docs/all", { allorgs: true });
    // To the current controller, this sends out
    // For REST environments, this would become: `/docs/all?allorgs=true` with a GET HTTP Method
    // For Meteor, this would become: `adaptivo/docs/all`
const documents = Doc.array().parse(response);
    // e.g. [ { ... }, { ... }, { ... } ]

// You can optionally disconnect from the server.
disconnect(server);
```

And using `@adaptivo/router` on the server, you could organise endpoints like so:

```javascript
import { createAuthenticatedRouter, createPublicRouter } from "@adaptivo/router";
// OR import { router } from "@adaptivo/feserver";
import { reify } from "@adaptivo/meteor/router";
import { DocsAll, DocsOne, DocsNew } from "./schema";
import { docs } from "./prisma-docs"
import { AuthenticatedContext } from "./context"

const router = createAuthenticatedRouter<User>({
    // Adapter should ensure User is not missing.
    'docs/new': async (ctx) => {
        const { userId, orgId } = await ctx.user
        const { content } = DocsNew.parse(ctx.query)
        return docs.create({
            data: {
                ownerId: userId,
                orgId: orgId,
                content: content,
            }
        })
    },

    'docs/one': async (ctx) => {
        const { userId, orgId } = await ctx.user
        const { id } = DocsOne.parse(ctx.query);
        return docs.findUnique({
            where: {
                id: id,
                OR: [
                    { ownerId: userId },
                    { orgId: orgId },
                ]
            }
        })
    },

    'docs/all': async (ctx) => {
        const { isSuperUser, orgId } = await ctx.user
        const { allorgs } = DocsAll.parse(ctx.query);
        if (!isSuperUser) {
            throw new Error("You are not a super user!")
        }
        if (allorgs) {
            return docs.findMany({
                where: { orgId: orgId }
            }) 
        }
        return docs.findMany() 
    },
})

// This gives Meteor all the routes and metadata and allows it to
// construct a series of Meteor methods (naturally this is irreversible!)
reify(router, {
    baseRoute: 'myapp', // all routes will now start with `myapp/{get,post}/...`
    extendContext: (ctx) => new AuthenticatedContext(ctx),
    // The context must provide a compatible `user: Promise<T>` type with router<T>
})

// Or a router can nest another router (but not itself, because then it'll become recursive)
router.add('nested', anotherRouter)
// This will produce the following routes:
// - myapp/docs/new
// - myapp/docs/one
// - myapp/docs/all
// - myapp/nested/*...


// Or you can use it programmatically... You just need to supply the right context!
const docsAll = router.get("docs/all")
```


