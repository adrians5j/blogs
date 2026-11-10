<!-- Draft. Publish once web functions are in a release users can install. Fill in the TODOs first. -->

<!-- TODO: link to the first post once it has a public URL. -->
In [our earlier post](./lambda-web-functions.md) we moved Webiny's API from API Gateway and a regular Lambda function to a Lambda web function, and measured GraphQL requests answering 25 to 48% faster at the median. <!-- TODO: the release, e.g. "Webiny 6.6" or "the first 6.6 beta". --> It's now in Webiny 6.6, and this post covers how to switch your project and what to expect when you do.

## New projects

New projects get web functions by default. There's nothing to set up.

## Existing projects

The upgrade adds one line to `webiny.config.tsx`:

```tsx
<Infra.Api.WebFunction enabled={false} />
```

That line keeps your current setup, API Gateway and all, exactly as it is. Upgrading Webiny shouldn't quietly change your infrastructure, so switching is your call.

When you're ready, set it to `true`, or delete the line, and deploy the API. We'd suggest trying it on a dev environment first and running your own tests against it, the heavy requests especially.

Setting it back to `false` and deploying again returns the project to API Gateway. <!-- TODO: confirm switching back was tested on a real project before publishing. -->

Your GraphQL schema, plugins, custom routes and extensions don't change. The URLs your apps call don't change either, because CloudFront stays in front of the API.

## VPC projects

Many teams run Webiny inside a VPC, often because OpenSearch lives there. A web function doesn't run inside your VPC. It reaches it through what AWS calls an egress network connector. Webiny creates one in the same private subnets and security group your other Lambda functions already use.

We tested that with a project that has both a VPC and OpenSearch. The web function wrote CMS entries and read them back through OpenSearch, which is only reachable from inside the VPC. Your VPC setup itself doesn't change. The subnets, the NAT gateway and the VPC endpoints stay as they are.

## Local development

`webiny watch` lets you run your API code on your own machine while the rest of the project stays deployed. It works the same way with web functions. While a session runs, requests to the web function are forwarded to your machine. We changed a route locally, called it through the deployed CloudFront URL, and got the new response without deploying anything.

## Tell us what you see

If you switch a project, we'd like to hear how it went, especially if your numbers don't look like ours.
