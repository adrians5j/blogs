Building on a modern foundation has always been important to us. When we first released Webiny back in 2020, we knew we wanted to create a platform that developers would actually enjoy working with. Fast forward to today, and we're excited to share what we've learned and how we've rebuilt Webiny from the ground up with version 6.

If you've been using Webiny v5, you know it's a powerful platform. But let's be honest - there were some pain points. The project structure was complex, onboarding new developers took time, and making changes sometimes felt slower than it should. We heard your feedback, and v6 is our answer.

In this article, I'll walk you through the key differences between v5 and v6. Whether you're considering upgrading or just getting started with Webiny, this should give you a clear picture of what's changed and why.

## The Philosophy Shift

Before diving into the technical details, it's worth understanding the fundamental shift in how we think about Webiny:

**v5 was:** "Here's everything, configure what you need"  
**v6 is:** "Here's what you need, extend when you want"

This philosophy is reflected in every decision we made. Let's see how.

## Simpler Project Structure

One of the most immediate differences you'll notice is the project structure.

**In v5**, you got a monorepo by default. Multiple workspaces, dozens of package.json files, scattered configuration files, and the infamous combo of `tsconfig.json` and `tsconfig.build.json`. Don't get me wrong - monorepos have their place, but for most Webiny projects, they added unnecessary complexity.

**In v6**, we've simplified everything:

- No monorepo by default
- Single root `package.json`
- Just one `tsconfig.json`
- Configuration lives in `webiny.config.tsx` and `extensions/`

This might seem like a small change, but it has a huge impact on daily development. Less context switching, less confusion, and a much clearer mental model of your project.

## Better Onboarding Experience

Remember your first time setting up a Webiny v5 project? It went something like this:

```bash
git clone ...
yarn
yarn setup-project
yarn build  # sometimes this was needed, sometimes it wasn't
```

And don't forget about the `.env` file that wasn't in git - easy to lose during onboarding, and if you lost those Pulumi secrets, you could break your deploys.

**In v6**, we've made onboarding trivial:

```bash
git clone ...
yarn
# That's it! You're ready to go.
```

The `.env` file is now optional. We don't store Pulumi secrets there for dev/non-prod environments. Everything just works out of the box.

## Modern Stack

We've upgraded the entire stack to use the latest and greatest technologies:

| Technology        | v5                  | v6                            |
| ----------------- | ------------------- | ----------------------------- |
| **Node.js**       | 18/20               | 24                            |
| **TypeScript**    | 4.x                 | 5.9                           |
| **Module System** | Mix of CJS and ESM  | Full ESM                      |
| **Build Tool**    | Webpack             | Rspack/Rsbuild (Rust-powered) |
| **Styling**       | Emotion (CSS-in-JS) | Tailwind v4                   |

The move to Rspack/Rsbuild deserves special attention. Builds are **dramatically faster** now. We're talking about the kind of speed improvement that changes how you work. No more waiting around for webpack to finish - your changes are reflected almost instantly.

## Professional Architecture with Dependency Injection

This is one of the biggest improvements, and it touches everything in the platform.

**v5 had no dependency injection system.** Dependencies were wired manually, patterns were inconsistent, and extending the platform meant digging through documentation to figure out which package to import and how to wire things up.

**v6 introduces a professional DI container** (`@webiny/di`) with:

- SOLID principles throughout the codebase
- Compile-time type safety
- The same DI pattern everywhere - backend, Admin, CLI, infrastructure
- Much cleaner APIs for extension development

This isn't just about code elegance (though it helps with that too). It's about making the platform easier to extend, customize, and maintain. When you want to swap out Cognito for Auth0 or Okta? It's now a one-line change. That's the power of proper dependency injection.

## Unified Storage with Headless CMS

In v5, each app had its own storage layer. Multiple abstractions, inconsistent APIs, and more complexity than necessary.

In v6, **everything is powered by Headless CMS.** One unified storage layer, consistent APIs across the entire platform. This simplification makes it easier to understand how data flows through your application and easier to build custom features on top of Webiny.

## Fewer Apps to Deploy

v5 shipped with 4 apps: Core, API, Admin, and Website. The Website app was mandatory, even if you just wanted to use Webiny as a headless CMS.

v6 reduces this to 3 apps: Core, API, and Admin. The Website is now optional - host it anywhere you want. And if you need advanced page building capabilities, we've built a completely new Website Builder app that you can add separately.

Less to deploy means faster deployments and lower infrastructure costs.

## Local Development That Actually Works

This one is a game-changer for daily development.

**In v5**, the watch command required redeploying Lambdas every time you made a change. Slow iteration, frustrating debugging, and a lot of waiting around.

**In v6**, we run Lambda functions locally by default. Make a change, refresh, see the result immediately. No redeployment needed. This turns the development cycle from minutes to seconds.

Try it out - add a breakpoint in your API code, start the dev server, and watch as you can debug your Lambda functions just like any other Node.js application. It's the kind of developer experience you'd expect from a modern framework.

## Flexible Database Configuration

v5 locked you into a database configuration per project. Want to use DynamoDB-only in dev and DynamoDB+OpenSearch in production? You'd have to recreate your project.

v6 lets you **choose your database configuration per environment:**

- DynamoDB-only in dev (faster, cheaper)
- DynamoDB+OpenSearch in prod (full search capabilities)
- Safety checks prevent accidental switches that could lose data

Speaking of OpenSearch, we've also upgraded from version 2.11 to 3.3, which brings better performance and is ready for semantic & AI search use cases.

## Structured Logging Everywhere

If you've ever tried to debug a production issue in v5 by looking at CloudWatch logs, you know the pain. Mostly unstructured logs, lots of plain text, hard to filter or search effectively.

v6 introduces **fully structured JSON logging everywhere:**

- Machine-friendly by default
- Fine-grained log levels
- Easy integration with observability tools like Datadog, New Relic, or CloudWatch Insights

This isn't just a nice-to-have. When something goes wrong in production, structured logs can save you hours of debugging time.

## Better CI/CD with OIDC

Security matters, and v5's approach to CI/CD wasn't ideal. You had to create IAM users, store long-lived AWS credentials in your CI secrets, and hope nobody compromised them.

v6 uses **OIDC-based authentication:**

- No AWS keys in your GitHub secrets
- Short-lived, scoped permissions
- CI/CD pipeline assumes a deployment role
- More secure by default

This follows AWS best practices and eliminates a whole class of security concerns.

## Extension System That Makes Sense

Both v5 and v6 support extensions, but v6 takes it to the next level.

We've created a dedicated repository at `github.com/webiny/extensions` where you can find well-documented, validated extensions that are easier to understand and maintain.

Some examples of what's possible:

- **Identity Providers:** Swap Cognito for Auth0 or Okta with a single line of code
- **Theming:** Change logos, colors, and tenant identity without hacking internals
- **Custom Page Types:** Build forms that capture meaningful business data, not just layout

The extension API is cleaner too. Less boilerplate, no manual DI imports, just your code doing what it needs to do.

## Single API Package

v5 exposed multiple `@webiny/*` packages, and discovering the right API for your use case often meant digging through incomplete documentation or asking on Discord.

v6 consolidates everything into a **single "webiny" package** with:

- Curated exports with structured paths
- Complete API reference
- Easy to discover what you need

Instead of juggling imports from a dozen different packages, you import from one place. Simple.

## Type Safety That Actually Works

v5 had TypeScript, but TypeScript errors didn't block builds. You could deploy broken code and only find out when it failed at runtime.

v6 makes TypeScript errors **fail the build.** If your types don't match, you can't deploy. This might sound strict, but it catches bugs before they reach production. Trust me, you want this.

## Easier Upgrades

Let's talk about the elephant in the room: upgrading between major versions.

v5 upgrades were complex. Lots of project-level code, many manual changes, breaking changes that required careful planning and execution.

v6 is designed for easier upgrades from the start:

- The platform handles more internally
- Less project-level code means fewer conflicts
- Simpler structure means simpler upgrade paths
- Low-friction by design

We can't promise zero-effort upgrades (that wouldn't be realistic), but we've removed a lot of the pain points that made v5 upgrades challenging.

## Reduced Package Count

This one might seem like an internal detail, but it matters:

- v5: ~160 internal `@webiny/*` packages
- v6: ~100 internal `@webiny/*` packages

Fewer packages means:

- Simpler dependency graph
- Lower maintenance overhead
- Faster installs
- Less chance of version conflicts

It's part of our overall simplification effort.

## What This All Means

If you're currently using v5, you might be wondering if upgrading is worth it. Here's my take:

**v6 is a better foundation in every way.** It's faster to work with, easier to understand, more secure, and built on modern technologies that will serve you well for years to come. The architecture is more professional, the developer experience is significantly improved, and the platform is more flexible.

That said, v5 isn't going anywhere immediately. We'll continue supporting it, but our focus is on making v6 the best it can be.

**If you're starting a new project**, use v6. No question about it.

**If you have an existing v5 project**, start planning your migration. The improvements are worth it, and we've built migration tools to help make the transition as smooth as possible.

## Conclusion

Webiny v6 represents everything we've learned from building and supporting v5 over the past several years. It's the result of listening to our community, understanding the pain points, and making deliberate architectural decisions to address them.

From the simplified project structure to the professional DI container, from local Lambda execution to structured logging, every change serves a purpose: making Webiny easier to use, faster to develop with, and more enjoyable overall.

I hope this article has given you a clear picture of what's changed and why. If you have questions or want to share your own experience migrating from v5 to v6, feel free to reach out to me on Twitter or join our community Discord.

Thanks for reading! My name is Adrian and I work as a full stack developer at Webiny. In my spare time, I like to write about my experiences with modern web development tools and frameworks, hoping it might help other developers. If you have any questions, comments, or just want to say hi, feel free to reach out!
