Last week we [shipped standalone Webiny](https://www.webiny.com/blog/webiny-is-no-longer-aws-only-introducing-standalone-webiny), the version you run on your own server instead of AWS.

From the outside that looks like a packaging job. Take the Lambda functions, run them in a Node process, point them at Postgres instead of DynamoDB, done.

Packaging was the small part. Most of the work went into the backend underneath, and this post is about that.

## The shortcut we didn't take

There was a quick way to do this, and we talked about it seriously.

You write a small Node server. It receives an HTTP request, dresses it up as an API Gateway event, and hands it to the same Lambda handler we already had. The code above that never finds out it isn't running on Lambda.

For plain HTTP requests, that works. It falls apart once you look at the rest, because a lot of what Webiny does on AWS leans on AWS services, not only on Lambda.

Background tasks are the clearest case. On AWS they run on Step Functions. A state machine starts the task, waits between steps and retries when one fails. A Node server has no Step Functions. You can fake the event a state machine would send to Lambda, but nothing on the other end is sending it. Something has to do the orchestration, and on a plain server that something doesn't exist until you write it.

Websockets are the same story. On AWS, API Gateway holds the connections. Our Lambda only sees connect, message and disconnect events, and to push something to a client it calls the API Gateway management API with a connection ID. On a Node server there's no API Gateway. The process holds the sockets itself, and sending a message means writing to a socket it has in memory. There's no event to fake, because the model is different.

So the wrapper would only have covered the easy part. For background tasks and websockets we'd have needed real server implementations anyway, and then we'd have hidden them behind fake AWS events so the code above could keep believing it was on AWS. Every new feature built for AWS first, then patched until the fakes held up. Two runtimes, one of them always a step behind.

We didn't want standalone to be the version that's always catching up. So we went the long way.

## What was actually down there

To see why the long way was long, you need to know how the Webiny backend was put together before.

Every request got a `context` object, and plugins filled it in. A plugin for the CMS would hang the CMS on it, the next plugin would read that and add its own thing, and so on down the list.

```ts
new ContextPlugin(async context => {
    context.cms = await createHeadlessCms(context);
});
```

That's simple, and it scaled a lot further than it had any right to. But it had two problems that got worse with every feature we added.

The first was order. `context.cms` only exists if the CMS plugin already ran, so a plugin that needs it has to be registered after it. With a handful of plugins you can keep that in your head. With the number we had, you can't, and the failure mode is a runtime error saying something "is not available" in a request that worked yesterday.

The second was that the whole thing was shaped like Lambda. The handler, the way events came in, the way requests were set up. Inside that Lambda handler sat Fastify, doing the HTTP routing. So an HTTP request on AWS went API Gateway, Lambda, Fastify, and finally our code. Moving that to a plain server means you're carrying Fastify around inside a structure that was only ever there because of Lambda.

Neither of those is something you fix with an adapter.

## A DI container instead of a context bag

The context object is gone. In its place is dependency injection, using [`@webiny/di`](https://www.npmjs.com/package/@webiny/di), our own DI package.

Every feature now registers what it provides and declares what it needs. The CMS doesn't go into a shared bag. It sits in the container as a recipe, and the container builds it the first time someone asks for it. The feature asking doesn't care who registered the CMS or when.

That fixed the ordering problem completely. You stop arranging plugins in the right sequence and start describing dependencies, and the container works out the sequence. I wrote about this in more detail a while back, because it was the change that surprised me most. A whole class of bug stopped existing and I didn't have to fix any of them individually.

It also sorted out something subtler, which is timing. A lot of backend services need to know the tenant and the current user before they can be built. In the plugin world they often got built before either of those was known, and then we worked around it. With the container they're built on first use, which in practice means after the request has been authenticated.

There's one catch. Resolving from the container is synchronous, so anything that needs an `await` to construct can't be built inside a lazy factory. We ended up splitting those services in two. The async part runs once, up front, and the lazy factory only puts the finished pieces together.

Each request gets its own child container, sitting on top of a root container that lives for the whole process. Things that are expensive and don't depend on who's asking live in the root. Things that do, live in the child and go away with the request. On Lambda that's one request per invocation. On a Node server it's many requests in one long-running process, and the same split works for both without anyone having to think about it.

Getting there meant moving every package off the old plugins, one PR at a time. The bar was zero `ContextPlugin` hits when you grep the repo. We got there, and then deleted the class.

## Fastify out

With DI in place, Fastify stopped earning its keep. We were using a fraction of it, and the fraction we used was fighting the container.

Routing is now our own `HttpRouter`, and a route is a regular class in the container like everything else:

```ts
class UploadFileRoute implements HttpRouteHandler.Interface {
    public constructor(private readonly config: FileManagerStandaloneConfig.Interface) {}

    public async handle(request: HttpRouteHandler.Request, response: HttpRouteHandler.Response) {
        if (!request.body) {
            return response.status(400).json({ error: "No file in request." });
        }
        // ...
    }
}
```

The route gets its dependencies through the constructor, same as any service, and gets a response object you'd recognise if you've written Express.

The router matches the URL before it builds anything, and only the route that matched gets constructed, along with whatever that one route depends on. A request for a file upload doesn't pay for the other routes' dependencies.

Owning the router also made some features cheap that would have been awkward before. Response compression, response streaming on both Lambda and the Node server, and routes having names so an extension can decorate one specific route.

## One core, two thin edges

This is the part the whole refactor was for.

The request handling now lives in one package, `event-handler-core`, and it has no idea where it's running. It takes an event, finds the handler for it, sets up the request container, runs the handler. That's the loop, and it's the same everywhere.

The platform-specific bits sit at the edges, and there are two of them. `event-handler-aws` knows about API Gateway, SQS, SNS, EventBridge, S3, DynamoDB streams and websocket events, and translates them into what the core expects. `event-handler-server` does the same for a plain Node HTTP server. The core's interface to them is small:

```ts
export interface Transport {
    bind(container: Container, ...rawArgs: any[]): void | Promise<void>;
}
```

A transport puts whatever platform-specific pieces it has into the request container. Everything after that is shared.

Before, "Lambda" was the shape of the whole backend. Now it's one of two adapters at the bottom, and the server version isn't pretending to be anything. It's a peer.

Above the transports, one function registers the whole feature set, and every transport calls it. AWS and the server can't drift apart, because there's only one list of features to keep up to date.

The pieces that really do depend on hosting plug in at named points: realtime, the scheduler and the file manager. The domain feature registers a Null default, and each transport replaces it with the real thing. On the server that's websockets held by the process, [Bree](https://github.com/breejs/bree) for scheduled jobs and the local disk for files. That's the answer to the Step Functions and API Gateway problem from earlier. Nothing gets faked. The server has its own implementations, sitting in the slots the AWS ones sit in.

Storage got the same treatment. DynamoDB and SQL are both storage implementations registered in the container, and the CMS asks for "storage" without knowing which one it got.

## What broke on the way

A refactor this size doesn't go cleanly, and the bugs it turned up taught us more than the parts that went smoothly. A few of them:

**Tenant before identity.** Requests authenticated with an API key were quietly coming through as anonymous. API key lookup is scoped to a tenant, so the tenant has to be known before the key can be checked. In the new setup it wasn't. The fix came down to the order of two decorators.

**Null defaults that say nothing.** For a while, pushing messages from the server to the browser over websockets didn't work at all. The connect event was never dispatched, so the Null websockets implementation stayed in place, and its `send()` did nothing without complaining. If that default had thrown an error, we'd have found it in minutes.

**License checks at registration time.** Some features checked the license inside `register()`. At that point the license hadn't been loaded yet, so they read the Null license, which always says no, and never registered. Nothing failed. The features just weren't there. We moved the license refresh to before registration.

**Installing from scratch.** The DI migration broke fresh installs. Before the install there's no root tenant, and the schema build that runs before the install expected one everywhere. Even the install request returned a 500. We now create a bootstrap root tenant, but only when it's missing.

**Background tasks that failed on success.** Background tasks on the new handler looked fine but didn't work. The handler returned nothing instead of the task result, and Step Functions marked every execution as FAILED.

The first three have the same shape. Something is missing, a default steps in, and nothing complains. Silent defaults felt safe when we added them, and they turned out to be the worst thing to debug.

## What it means if you build on Webiny

Your extensions don't know which platform they're on either.

A custom GraphQL field, a lifecycle hook, a new HTTP route, a background task. You write it against the container, and it runs the same on Lambda and on a Node server. You don't keep two versions, and there's nothing to port when you move a project from one to the other. That's what I meant in the launch post when I said the APIs don't change. Keeping them the same took no extra effort, because there's only one core underneath.

It helped us too. Our tests go through the same handler loop production does, instead of a hand-assembled context, so they're a lot closer to what actually runs.

## Was it worth it

It cost us a big chunk of this year, and a lot of that time went into work that nobody sees in the admin UI. There were weeks where the whole diff was "this still does exactly what it did, but it's wired differently now." That's not a fun changelog to write.

But the result is that standalone isn't a second product we have to keep in sync. There's one backend, and it happens to run in two places. Adding a third place, if we ever want one, is a new edge and not another rewrite.

If you run into something on standalone that behaves differently from AWS, I'd like to hear about it. With this design it shouldn't happen, so it's a bug.
