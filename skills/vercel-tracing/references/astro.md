# Astro instrumentation

Use this reference when the application uses Astro with `@astrojs/vercel`. Check [SDK ownership](instrumentation.md) before registering OpenTelemetry.

Astro emits no framework spans and does not extract incoming trace context. It does not load an `instrumentation.ts` file. Custom spans stay in a separate trace unless you extract the incoming headers before `next()`.

Initialize Vercel's SDK in `src/telemetry.ts`:

```ts
import { registerOTel } from '@vercel/otel';

registerOTel({ serviceName: 'checkout' });
```

Astro loads `src/middleware.ts` or `src/middleware.js`, and falls back to `src/middleware/index.ts` only when neither exists. Use the file the app already has. Creating `src/middleware.ts` next to `src/middleware/index.ts` stops the existing middleware from loading, with no warning. Import the telemetry module first in that file, as `'../telemetry'` from `src/middleware/index.ts`. The import runs before `next()`, and Node loads the module once per function instance. Import OpenTelemetry's context as `otelContext`. Astro's middleware argument is also named `context`.

```ts
import './telemetry';
import { context as otelContext, propagation } from '@opentelemetry/api';
import { defineMiddleware } from 'astro:middleware';

export const onRequest = defineMiddleware((context, next) => {
  if (context.isPrerendered) return next();
  const parent = propagation.extract(
    otelContext.active(),
    Object.fromEntries(context.request.headers),
  );
  return otelContext.with(parent, () => next());
});
```

If the middleware file already exports `onRequest`, keep that handler and call it inside `otelContext.with`. Do not add `src/fetch.ts` only to register OpenTelemetry. Astro 7 uses that file as the request pipeline.

Leave `middlewareMode` unset and remove `edgeMiddleware: true`. Either `middlewareMode: 'edge'` or the older `edgeMiddleware: true` deploys this file as an Edge function, and the Edge runtime does not support custom spans. See [Astro middleware](https://docs.astro.build/en/guides/middleware/) and the [Vercel adapter](https://docs.astro.build/en/guides/integrations-guide/vercel/).

Traced routes must render on demand. Set `export const prerender = false` on the route, or set `output: 'server'` to server-render every page. Prerendered pages produce no application spans. Middleware also runs while those pages prerender, so return `next()` when `context.isPrerendered` is true. ISR cache hits do not run this middleware, so they produce no application spans.

Add operation spans as in [custom spans](custom-spans.md).

Deploy the change to preview or production, then verify [collection](collection.md) and exercise an on-demand route.
