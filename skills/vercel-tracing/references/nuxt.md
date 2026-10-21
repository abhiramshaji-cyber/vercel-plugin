# Nuxt instrumentation

Use this reference when the application uses Nuxt. Check [SDK ownership](instrumentation.md) before registering OpenTelemetry.

Nuxt 4 builds its server with Nitro 2. Nitro 2 emits no framework spans and does not extract incoming trace context. Register the SDK in `server/plugins/telemetry.ts`. Nitro runs server plugins once per instance when the server module loads, before lazily loaded route handlers. Nuxt auto-imports `defineNitroPlugin` and `getRequestHeaders` in `server/`.

```ts
import { registerOTel } from '@vercel/otel';
import { context, propagation } from '@opentelemetry/api';

export default defineNitroPlugin((nitroApp) => {
  registerOTel({ serviceName: 'checkout' });
  const handler = nitroApp.h3App.handler;
  nitroApp.h3App.handler = (event) => {
    const parent = propagation.extract(context.active(), getRequestHeaders(event));
    return context.with(parent, () => handler(event));
  };
});
```

A `request` hook and server middleware return before the route handler and the page render. `context.with` in those hooks does not cover later spans. Replacing `h3App.handler` runs API routes and server-rendered pages inside the extracted context. See [Nuxt server plugins](https://nuxt.com/docs/4.x/directory-structure/server#server-plugins) and [incoming requests](context-propagation.md#incoming-requests).

Put the `measure` helper from [custom spans](custom-spans.md) in `server/utils/measure.ts`. Nitro auto-imports it in server routes. Do not import that file from a Vue page. A server-rendered page that calls a relative API route with `useFetch` runs that route in-process. Its spans join the page request under the function span. See [Nuxt server utilities](https://nuxt.com/docs/4.x/directory-structure/server#server-utilities).

Keep the default `vercel` preset. It runs on Node.js. Do not set `nitro.preset` to `vercel-edge`. Custom spans need the Node.js runtime. See [custom spans](custom-spans.md). Prerendered routes run no server code. Do not set `isr` or `swr` on a route you need to trace. Nitro 2 maps a `swr` route rule to `isr` on Vercel unless `future.nativeSWR` is set. Vercel serves an ISR cache hit without running the handler.

Nuxt 4 does not ship Nitro 3. Do not set `tracingChannel`.

Deploy the change to preview or production, then verify [collection](collection.md) and exercise a dynamic server-rendered route.
