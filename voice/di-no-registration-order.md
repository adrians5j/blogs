# With DI, you stop babysitting registration order

Here's a small thing that clicked for me while refactoring our framework to be
dependency-injection-native — one of those changes that feels like nothing until you realize how
much mental overhead it quietly removes.

Before, in the old plugin/context world, the **order in which you registered things mattered**. A
lot. We had these per-request "context initializers" that ran in sequence, and each one would
eagerly build stuff that the next ones depended on. The CMS initializer had to run before the ACO
one, which had to run before the headless-cms-tasks one, and so on. If you got the order wrong — or
a teammate added a new feature in the wrong spot — something downstream would blow up with a cryptic
"X is not available" at runtime. So you were always holding this fragile ordering in your head, like
a little dependency graph you had to topologically sort by hand.

The thing I kept doing was *watching the order*. Register A, then B, then C — because C reads
something A set up.

With DI, that whole concern just... evaporates. And the mechanism that makes it evaporate is worth
understanding, because it's the actual "aha": **lazy factories**.

## Eager vs. lazy

There are two ways to put something into the DI container.

The first is **eager** — you build the thing right now and hand the finished object to the container:

```ts
const cms = buildTheWholeFacade();        // built RIGHT NOW
container.registerInstance(HeadlessCms, cms);
```

The object exists the moment you register it, whether or not anyone ends up needing it. This is what
forced the ordering on us: if building the CMS facade happens *now*, then everything that reads the
CMS facade has to be arranged to run *after* now.

The second is a **lazy factory** — you don't build anything, you just hand the container a *recipe*:

```ts
container.registerFactory(HeadlessCms, () => buildTheWholeFacade());   // just a recipe
```

The container holds onto that function and does nothing with it. It only runs it the first time
someone actually asks for the thing — `container.resolve(HeadlessCms)` — and then caches the result
so everyone after that gets the same instance.

"Lazy" = built on first use, not at registration time. "Factory" = the build-function the container
is holding.

## Why this kills the ordering problem

Once registration is just "pin a recipe to the fridge," the order you pin them in is irrelevant. A
can register its recipe before or after B. Whoever calls `resolve()` first triggers the build, and
the container automatically resolves whatever *that* recipe depends on, recursively. You stopped
describing *steps in sequence* and started describing *dependencies* — and the container does the
sequencing for you.

There's a second, quieter win hiding in the timing. In a web request the lifecycle is roughly:

```
register()  ──►  auth/tenant established  ──►  resolvers run
```

The CMS facade needs the tenant (to load the right storage) and the identity (for access control).
With eager `registerInstance`, you're forced to build it during `register()` — *before* you even
know who the user is. With a lazy factory, the build is deferred until something resolves it, which
happens during the resolvers — *after* auth has run. So "lazy" doesn't just decouple order, it also
lets the thing get built late enough to actually have the information it needs.

## The catch: lazy factories resolve *synchronously*

It's not all free, and the place it bit me is worth writing down, because it's the kind of thing
that's obvious in hindsight and completely invisible up front.

Our DI container resolves synchronously. When you write:

```ts
const cms = container.resolve(HeadlessCms);
```

there's no `await` there — and there *can't* be, because half the callers are plain synchronous
code. Which means a lazy factory has to build its thing **synchronously too**. The recipe runs, start
to finish, in one tick.

That's fine until the thing you're building needs an `await` to construct. Mine did: the CMS facade
is assembled on top of the storage layer, and creating the storage layer was an `async` operation.
So I couldn't just drop the whole facade into a lazy factory — the factory would need to `await`, and
it can't.

The fix is to split the work by *how* it's built, not by what it is:

- The **async-to-construct** part (build the storage layer) stays **eager** — it runs once, per
  request, in a small hook that fires before any resolver. `await` is allowed there.
- The **synchronous assembly** (wire the storage layer + permissions + the CRUD methods into the
  facade object) becomes the **lazy factory**. By the time anyone resolves it, the async part it
  depends on is already sitting in the container, so the assembly is pure synchronous glue.

So the facade is still lazy and still order-independent — I just had to recognize that "lazy" only
applies to things you can construct in one synchronous breath. Anything with an `await` in its
construction has to be built ahead of time and *handed* to the lazy factory, not built *inside* it.

The general shape, once it clicked: **async construction and synchronous resolution are an impedance
mismatch, and you design around it by separating the async build from the sync assembly.** Miss that,
and you reach for "just make the whole thing a factory" and get stuck wondering why your factory
wants to be async when the container won't let it.

## The mental model

The way I think about it now:

- `registerInstance` is cooking the dish and leaving it on the counter in case someone's hungry.
- `registerFactory` is pinning the recipe to the fridge — nobody cooks until someone actually orders,
  and by then you know how many guests (the tenant) and their allergies (the identity).

The surprising part wasn't that the code got cleaner. It's that an entire *category of bug* — and an
entire thing I had to keep in my head — simply stopped existing. You describe the dependencies; the
container handles the order.
