# Webiny v5 vs v6 - Complete Comparison Table

## Architecture & Code Quality

| Category                 | Webiny v5                                                                           | Webiny v6                                                                                                                                                       |
| ------------------------ | ----------------------------------------------------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Dependency Injection** | ❌ No DI system<br>Manual dependency wiring<br>Inconsistent patterns                | ✅ Professional DI container (@webiny/di)<br>SOLID principles throughout<br>Compile-time type safety<br>Same DI pattern everywhere (backend, Admin, CLI, infra) |
| **Internal Packages**    | ~160 @webiny/\* packages<br>Complex dependency graph<br>Higher maintenance overhead | ~100 @webiny/\* packages<br>Simplified architecture<br>Better maintainability                                                                                   |
| **Storage Architecture** | Each app has own storage layer<br>Multiple abstractions<br>Inconsistent APIs        | Everything powered by Headless CMS<br>Unified storage layer<br>Consistent APIs across platform                                                                  |

## Developer Experience

| Category                  | Webiny v5                                                                                                                                   | Webiny v6                                                                                                                                      |
| ------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------- |
| **Project Structure**     | Monorepo by default<br>Multiple workspaces<br>Many package.json files<br>Scattered configuration<br>`tsconfig.json` + `tsconfig.build.json` | No monorepo by default<br>`webiny.config.tsx` + `extensions/`<br>Single root package.json<br>Centralized configuration<br>Just `tsconfig.json` |
| **Configuration**         | Required `.env` file (not in git)<br>Easy to lose on onboarding<br>Pulumi secrets in `.env`<br>Could break deploys if lost                  | `.env` optional<br>No Pulumi secrets in `.env` for dev/non-prod<br>Declarative `webiny.config.tsx`<br>React components for config              |
| **Project Onboarding**    | `git clone`<br>`yarn`<br>`yarn setup-project`<br>`yarn build` (sometimes)                                                                   | `git clone`<br>`yarn`<br>✅ Ready!                                                                                                             |
| **API Package**           | Multiple @webiny/\* packages<br>Scattered APIs<br>Hard to discover<br>Incomplete docs                                                       | Single "webiny" package<br>Curated exports with structured paths<br>Easy to discover<br>Complete API reference                                 |
| **Extension Development** | Required @webiny/di imports<br>Manual `createImplementation` calls<br>More boilerplate                                                      | Cleaner API<br>No manual DI imports needed<br>Less boilerplate, just your code                                                                 |

## Project Apps & Deployment

| Category           | Webiny v5                                                                       | Webiny v6                                                                                                                                 |
| ------------------ | ------------------------------------------------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------- |
| **App Structure**  | 4 apps: Core, API, Admin, Website<br>Website mandatory<br>More to deploy/manage | 3 apps: Core, API, Admin<br>Website optional (host anywhere)<br>New Website Builder app (separate)                                        |
| **CloudFormation** | IAM users + user groups<br>Long-lived credentials<br>Manual user setup          | Managed IAM policies + deployment role<br>CI/CD assumes role<br>OIDC for GitHub Actions<br>No AWS keys in secrets<br>Built for automation |

## Modern Stack & Build Tools

| Category               | Webiny v5                                                  | Webiny v6                                                                                            |
| ---------------------- | ---------------------------------------------------------- | ---------------------------------------------------------------------------------------------------- |
| **Runtime & Language** | Node.js 18/20<br>Mix of CommonJS and ESM<br>TypeScript 4.x | Node.js 24<br>Full ESM<br>TypeScript 5.9                                                             |
| **Build Tools**        | Webpack<br>CRA legacy code<br>Slower builds                | Rspack/Rsbuild (Rust-powered)<br>CRA code removed<br>Dramatically faster builds<br>Cleaner summaries |
| **UI Framework**       | Tailwind v3                                                | Tailwind v4<br>Smaller bundles<br>Own design system with Storybook                                   |

## Database & Infrastructure

| Category                 | Webiny v5                                                                                  | Webiny v6                                                                                                          |
| ------------------------ | ------------------------------------------------------------------------------------------ | ------------------------------------------------------------------------------------------------------------------ |
| **Database Flexibility** | Fixed DB setup per project<br>Recreate project to change DB config<br>One size fits all    | Choose DB per environment<br>DynamoDB-only in dev<br>DynamoDB+OpenSearch in prod<br>Safety checks prevent switches |
| **OpenSearch**           | OpenSearch 2.11                                                                            | OpenSearch 3.3<br>Ready for semantic & AI search<br>Faster queries<br>Better similarity searches                   |
| **Audit Logs**           | Same DynamoDB table as app data<br>Mixed with application data<br>More noise in main table | Dedicated audit logs table<br>Cleaner data separation<br>Better data hygiene                                       |

## Development Workflow

| Category             | Webiny v5                                                                             | Webiny v6                                                                                                    |
| -------------------- | ------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------ |
| **API Development**  | Watch command requires redeploying Lambdas<br>Slow iteration<br>Frustrating debugging | Local Lambda execution by default<br>Instant changes without redeploy<br>Fast iteration<br>Easy debugging    |
| **Build Validation** | TypeScript errors don't block builds<br>Could deploy broken code                      | TypeScript error blocking<br>Build fails if types don't match<br>Catch errors before deploy                  |
| **CLI Experience**   | `--env` flag required<br>More verbose commands                                        | `--env` defaults to "dev"<br>Cleaner, shorter commands<br>Improved CLI output<br>Structured logs in terminal |

## Observability & Logging

| Category              | Webiny v5                                                            | Webiny v6                                                                                                                                               |
| --------------------- | -------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Logging Structure** | Mostly unstructured<br>Hard to work with<br>Plain text in CloudWatch | Fully structured everywhere<br>JSON logs in CloudWatch<br>Machine-friendly by default<br>Fine-grained log levels<br>Easy observability tool integration |

## Extensions & Customization

| Category               | Webiny v5                                                                                 | Webiny v6                                                                                                                                                                           |
| ---------------------- | ----------------------------------------------------------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Extension System**   | `webiny extension` command<br>Less clear structure                                        | `webiny extension` command (still available!)<br>Dedicated repo: github.com/webiny/extensions<br>Easier to understand/extend/maintain<br>Strict validation with compile-time checks |
| **Identity Providers** | Built-in, harder to swap<br>Deep configuration needed                                     | Identity providers are extensions<br>Swap Cognito for Auth0/Okta (one line)<br>No rewiring                                                                                          |
| **Theming**            | Complex tenant-level theming<br>Required hacks and overrides<br>Digging through internals | Simple, explicit extension<br>Logos, colors, tenant identity in one place<br>Real React components<br>No hacks                                                                      |
| **Page Builder**       | Standard Page Builder<br>Limited customization                                            | Completely new product<br>Custom page types<br>Full control over creation forms<br>Meaningful business data<br>Customize editor itself<br>Control data access & API responses       |

## CI/CD & Testing

| Category           | Webiny v5                                                   | Webiny v6                                                                                                        |
| ------------------ | ----------------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------- |
| **CI Testing**     | Auto-install not straightforward<br>Awkward CI setups       | Single component handles auto-install<br>Conditional auto-install<br>CI-friendly by design                       |
| **GitHub Actions** | Long-lived AWS credentials in secrets<br>Static access keys | OIDC-based authentication<br>Short-lived, scoped permissions<br>No AWS keys in secrets<br>More secure by default |

## Upgrades & Maintenance

| Category               | Webiny v5                                                                                                  | Webiny v6                                                                                                                                                                |
| ---------------------- | ---------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| **Upgrade Difficulty** | Complex upgrades<br>Lots of project-level code<br>Many manual changes<br>Breaking changes harder to handle | Much easier upgrades<br>Platform handles more internally<br>Less project-level code<br>Fewer manual changes<br>Simpler structure = simpler upgrades<br>Low-friction path |

---

## Quick Reference Table

| Feature                  | v5                               | v6                           |
| ------------------------ | -------------------------------- | ---------------------------- |
| **Architecture**         | No DI, scattered                 | DI container, SOLID          |
| **Project Structure**    | Monorepo                         | Simple (config + extensions) |
| **Node.js**              | 18/20                            | 24                           |
| **TypeScript**           | 4.x                              | 5.9                          |
| **Module System**        | CJS + ESM                        | Full ESM                     |
| **Build Tool**           | Webpack                          | Rspack/Rsbuild               |
| **UI Framework**         | Tailwind v3                      | Tailwind v4                  |
| **Apps to Deploy**       | 4 (Core, API, Admin, Website)    | 3 (Core, API, Admin)         |
| **OpenSearch**           | 2.11                             | 3.3                          |
| **Database Flexibility** | Fixed per project                | Per environment              |
| **Logging**              | Unstructured                     | Structured (JSON)            |
| **API Package**          | Multiple @webiny/\* packages     | Single "webiny" package      |
| **Onboarding**           | git clone → yarn → setup → build | git clone → yarn             |
| **Config Files**         | .env required, 2 tsconfigs       | .env optional, 1 tsconfig    |
| **IAM for Deploy**       | Users + groups                   | Roles + OIDC                 |
| **Extensions**           | Available                        | Enhanced + validated         |
| **Internal Packages**    | ~160                             | ~100                         |
| **CI Testing**           | Awkward                          | Simple                       |
| **Upgrades**             | Complex                          | Easy                         |
| **Type Safety**          | Runtime only                     | Compile-time                 |
| **API Development**      | Redeploy each change             | Local Lambda execution       |

---

## Key Improvements in v6

| Improvement Area              | Impact                                                                        |
| ----------------------------- | ----------------------------------------------------------------------------- |
| **Simpler Everything**        | From project structure to configuration to onboarding                         |
| **Modern Foundation**         | Latest Node, full ESM, TypeScript 5.9, Rust-powered builds                    |
| **Better DX**                 | Faster builds, easier debugging, clearer errors, better docs                  |
| **More Flexible**             | Per-environment DB configs, optional components, extension-based architecture |
| **More Secure**               | OIDC, role-based deploys, no static credentials                               |
| **Better Observability**      | Structured logging everywhere, JSON in CloudWatch                             |
| **Easier Maintenance**        | Less code, clearer patterns, easier upgrades                                  |
| **Professional Architecture** | DI container, SOLID principles, type-safe at compile time                     |

---

## Philosophy Shift

| Version | Philosophy                                   |
| ------- | -------------------------------------------- |
| **v5**  | "Here's everything, configure what you need" |
| **v6**  | "Here's what you need, extend when you want" |

**v6 is about removing friction, improving clarity, and building on a rock-solid modern foundation.**

---

_Last updated: January 2026_
