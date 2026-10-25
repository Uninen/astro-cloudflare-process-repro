# Cloudflare 14.4.0 breaks a client-only Vue island

Uses Astro 7.4.0, Vue 3.5.44, `@astrojs/vue` 7.1.0, `@vuetify/v0` 1.2.3, and
`@astrojs/cloudflare` 14.4.0. The page contains one Vue component with one Vuetify button.

## Reproduce

Requires Node.js 22.22.0 or newer and pnpm.

```sh
pnpm install
pnpm build
pnpm preview
```

Open <http://localhost:4321/>. Expected: a button labelled `Vue rendered`.
Actual: a blank page. The browser console reports an Astro hydration error containing
`ReferenceError: process is not defined`. The production build itself succeeds.

## Cause

In production, `@vuetify/v0@1.2.3` reads `process.env.VITE_LOGGER_ENABLED` during module loading.
Browsers do not provide Node.js's `process` object.

Vite normally replaces `process.env` with `{}` in browser bundles. The Cloudflare adapter instead sets
`vite.define['process.env'] = 'process.env'` on the shared build configuration. This tells Vite to preserve
that reference in browser code too.

Adapter 14.3.3 masked this configuration leak by injecting a dummy `process.env` object into browser
bundles. In 14.4.0, [PR #18213](https://github.com/withastro/astro/pull/18213) restricted that shim to the
server environments but left the define global. Vuetify's remaining reference now throws before Vue
can mount the component.

## Control

Stop the preview, change only the adapter version, and rebuild:

```sh
pnpm preview stop
pnpm add --save-exact @astrojs/cloudflare@14.3.3
pnpm build
pnpm preview
```

The button renders without browser errors. Both versions were verified locally in Firefox.
