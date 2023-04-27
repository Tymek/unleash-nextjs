> ## 🏗️ Early version
>
> We need your feedback. Share your comments in [🗣️ GitHub Discussions](https://github.com/Unleash/unleash/discussions) or on Unleash community [💬 Slack](https://unleash-community.slack.com/) server.

# Unleash Next.js SDK

This package allows easy integration of [Unleash](https://github.com/unleash/unleash) feature flags in a [Next.js](https://nextjs.org/) application.

## Setup

### Installation

To install, simply run:

```sh
npm install @unleash/nextjs
# or
yarn add @unleash/nextjs
# or
pnpm add @unleash/nextjs
```

### Environment variables

This package will attempt to load configuration from
[Next.js Environment variables](https://nextjs.org/docs/basic-features/environment-variables).

When using Unleash **client-side**, with `<FlagProvider />` or `getFrontendFlags()` configure:

- `NEXT_PUBLIC_UNLEASH_FRONTEND_API_URL`. URL should end with `/api/frontend` or `/proxy`
- `NEXT_PUBLIC_UNLEASH_FRONTEND_API_TOKEN` [client-side API token](https://docs.getunleash.io/reference/api-tokens-and-client-keys#front-end-tokens)
  if you're using the [front-end API](https://docs.getunleash.io/reference/front-end-api),
  or a [proxy client key](https://docs.getunleash.io/reference/api-tokens-and-client-keys#proxy-client-keys)
  if you're using a [proxy](https://docs.getunleash.io/reference/unleash-proxy)

If using **server-side** (SSR, SSG, API), using `getDefinitions()` and `evaluateFlags()`, set:

- `UNLEASH_SERVER_API_URL` of you instance. URL should end with `/api`
- `UNLEASH_SERVER_API_TOKEN` [server-side API client token](https://docs.getunleash.io/reference/api-tokens-and-client-keys#client-tokens)

#### Detailed explanation

| Prefixable     | Variable                     | Default                                                   |
| -------------- | ---------------------------- | --------------------------------------------------------- |
| `NEXT_PUBLIC_` | `UNLEASH_SERVER_API_URL`     | `http://localhost:4242/api`                               |
| `NEXT_PUBLIC_` | `UNLEASH_FRONTEND_API_URL`   | `<(NEXT_PUBLIC_)UNLEASH_SERVER_API_URL>/frontend`         |
| **No**         | `UNLEASH_SERVER_API_TOKEN`   | `default:development.unleash-insecure-api-token`          |
| `NEXT_PUBLIC_` | `UNLEASH_FRONTEND_API_TOKEN` | `default:development.unleash-insecure-frontend-api-token` |
| `NEXT_PUBLIC_` | `UNLEASH_APP_NAME`           | `nextjs`                                                  |

If you plan to use configuration in the browser, add `NEXT_PUBLIC_` prefix.
If both are defined and available, private variable takes priority.
You can use both to have different values on client-side and server-side.

# Usage

## A). Client-side only - simple use case and for development purposes (CSR)

Fastest way to get started is to connect frontend directly to Unleash.
You can find out more about direct [Front-end API access](https://docs.getunleash.io/reference/front-end-api) in our documentation,
including a guide on how to [setup a client-side SDK key](https://docs.getunleash.io/how-to/how-to-create-api-tokens).

```tsx
import type { AppProps } from "next/app";
import { FlagProvider } from "@unleash/nextjs";

export default function App({ Component, pageProps }: AppProps) {
  return (
    <FlagProvider>
      <Component {...pageProps} />
    </FlagProvider>
  );
}
```

With `<FlagProvider />` in place you can now use hooks like: `useFlag`, `useVariant`, or `useFlagsStatus` to block rendering until flags are ready.

```jsx
import { useFlag } from "@unleash/nextjs";

const YourComponent = () => {
  const isEnabled = useFlag("nextjs-example");

  return <>{isEnabled ? "ENABLED" : "DISABLED"}</>;
};
```

Optionally, you can configure `FlagProvider` with the `config` prop. It will take priority over environment variables.

```jsx
<FlagProvider
  config={{
    url: "http://localhost:4242/api/frontend", // replaces NEXT_PUBLIC_UNLEASH_FRONTEND_API_URL
    clientKey: "<Frontend_API_token>", // replaces NEXT_PUBLIC_UNLEASH_FRONTEND_API_TOKEN
    appName: "nextjs", // replaces NEXT_PUBLIC_UNLEASH_APP_NAME

    refreshInterval: 15, // additional client configuration
    // see https://github.com/Unleash/unleash-proxy-client-js#available-options
  }}
>
```

If you only plan to use [Unleash client-side React SDK](https://github.com/Unleash/proxy-client-react) now also works with Next.js. Check documentation there for more examples.

## B). Static Site Generation, optimized performance (SSG)

With same access as in the client-side example above you can resolve Unleash feature flags when building static pages.

```tsx
import {
  flagsClient,
  getDefinitions,
  evaluateFlags,
  getFrontendFlags,
  type IVariant,
} from "@unleash/nextjs";
import type { GetStaticProps, NextPage } from "next";

type Data = {
  isEnabled: boolean;
  variant: IVariant;
};

const ExamplePage: NextPage<Data> = ({ isEnabled, variant }) => (
  <>
    Flag status: {isEnabled ? "ENABLED" : "DISABLED"}
    <br />
    Variant: {variant.name}
  </>
);

export const getStaticProps: GetStaticProps<Data> = async (_ctx) => {
  /* Using server-side SDK: */
  const definitions = await getDefinitions();
  const context = {}; // optional, see https://docs.getunleash.io/reference/unleash-context
  const { toggles } = evaluateFlags(definitions, context);

  /* Or with the proxy/front-end API */
  // const { toggles } = await getFrontendFlags({ context });

  const flags = flagsClient(toggles);

  return {
    props: {
      isEnabled: flags.isEnabled("nextjs-example"),
      variant: flags.getVariant("nextjs-example"),
    },
  };
};

export default ExamplePage;
```

The same approach will work for [ISR (Incremental Static Regeneration)](https://nextjs.org/docs/basic-features/data-fetching/incremental-static-regeneration).

Both `getDefinitions()` and `getFrontendFlags()` can take arguments overriding URL, token and other request parameters.

## C). Server Side Rendering (SSR)

```tsx
import {
  flagsClient,
  evaluateFlags,
  getDefinitions,
  type IVariant,
} from "@unleash/nextjs";
import type { GetServerSideProps, NextPage } from "next";

type Data = {
  isEnabled: boolean;
};

const ExamplePage: NextPage<Data> = ({ isEnabled }) => (
  <>Flag status: {isEnabled ? "ENABLED" : "DISABLED"}</>
);

export const getServerSideProps: GetServerSideProps<Data> = async (ctx) => {
  const sessionId =
    ctx.req.cookies["unleash-session-id"] ||
    `${Math.floor(Math.random() * 1_000_000_000)}`;
  ctx.res.setHeader("set-cookie", `unleash-session-id=${sessionId}; path=/;`);

  const context = {
    sessionId, // needed for stickiness
    // userId: "123" // etc
  };

  const { toggles } = await getFrontendFlags({ context }); // Use Proxy/Frontend API
  const flags = flagsClient(toggles);

  return {
    props: {
      isEnabled: flags.isEnabled("nextjs-example"),
    },
  };
};

export default ExamplePage;
```

## D). Bootstrapping / rehydration

You can bootstrap Unleash React SDK to have values loaded from the start.
Initial value can be customized server-side.

```tsx
import App, { AppContext, type AppProps } from "next/app";
import {
  FlagProvider,
  getFrontendFlags,
  type IMutableContext,
  type IToggle,
} from "@unleash/nextjs";

type Data = {
  toggles: IToggle[];
  context: IMutableContext;
};

export default function CustomApp({
  Component,
  pageProps,
  toggles,
  context,
}: AppProps & Data) {
  return (
    <FlagProvider
      config={{
        bootstrap: toggles,
        context,
      }}
    >
      <Component {...pageProps} />
    </FlagProvider>
  );
}

CustomApp.getInitialProps = async (ctx: AppContext) => {
  const context = {
    userId: "123",
  };

  const { toggles } = await getFrontendFlags(); // use Unleash Proxy

  return {
    ...(await App.getInitialProps(ctx)),
    bootstrap: toggles,
    context, // pass context along so client can refetch correct values
  };
};
```

## E). Middleware

It's possible to run this SDK in Next.js Edge Middleware. This is a great use case for A/B testing, where you can transparently redirect users to different pages based on a feature flag. Target pages can be statically generated, improving performance.

See [`./example/README.md#Middleware`](https://github.com/Unleash/unleash-client-nextjs/blob/main/example/README.md#middleware)

# ⚗️ CLI (experimental)

You can use `unleash [action] [options]` in your `package.json` `scripts` section, or with:

```sh
npx @unleash/nextjs
```

For the CLI to work, the following [environment variables](#environment-variables) must be configured with appropriate values:

- `UNLEASH_SERVER_API_URL`
- `UNLEASH_SERVER_API_TOKEN`

The CLI will attempt to read environment values from any `.env` files if they're present. You can also set the variables directly when invoking the interface, as in the [CLI usage example](#example).

## Usage

- `get-definitions <outputFile.json>` Download feature flags definitions for bootstrapping (offline use) of server-side SDK.
- `generate-types [options] <outputFile.ts>` Generate types and typed functions from feature flags defined in an Unleash instance.
  It will also generate strictly typed versions of `useFlag`, `useVariant`, `useFlags` and `flagsClient` (unless `--typesOnly` is used).
  - `-t, --typesOnly` don't include typed versions of functions exported from `@unleash/nextjs` (default: false)
  - `-b, --bootstrap <sourceFile.json>` load definitions from a file instead of fetching definitions - work offline
- `-V` Output the version number

## Example

Try it now

```sh
UNLEASH_SERVER_API_URL=https://app.unleash-hosted.com/demo/api \
UNLEASH_SERVER_API_TOKEN=test-server:default.8a090f30679be7254af997864d66b86e44dcfc5291916adff72a0fb5 \
npx @unleash/nextjs generate-types ./unleash.ts
```

# What's next

## Experimental features support

Unleash Next.js SDK can run on [Edge Runtime](https://nextjs.org/docs/api-reference/edge-runtime) and in [Middleware](https://nextjs.org/docs/advanced-features/middleware). We are also interested in providing an example with [App Directory](https://beta.nextjs.org/docs/app-directory-roadmap).

## Known limitation

- In current interation server-side SDK does not support metrics.
- When used server-side, this SDK does not support the "Hostname" and "IP" strategies. Use custom context fields and constraints instead.
