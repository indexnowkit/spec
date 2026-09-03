# 36. JS/TS: `@indexnowkit/sveltekit`, `@indexnowkit/react-router`

Оба фреймворка без фонового примитива. Пакеты маленькие: key route + dispatcher по платформе +
хелперы для actions.

## SvelteKit

- `src/routes/[key].txt/+server.ts`: `export const GET = keyFileHandler()`.
- `hooks.server.ts`: `handle = sequence(indexNowHandle(), ...)` открывает Collector-область и
  после `resolve(event)` делает flush через `event.platform?.context?.waitUntil` (Cloudflare,
  `app.d.ts` типы) или `waitUntil` из `@vercel/functions` (adapter-vercel), иначе `await`
  перед возвратом (документировано как добавление задержки ≤ timeout; рекомендация sync timeout 3 s).
- В `+page.server.ts` actions: `await indexNow.submit(...)` или ORM-адаптер.

## React Router 7 (framework mode) / Remix

- Resource route `app/routes/[key].txt.ts` с `loader` → `keyFileLoader()`.
- `getLoadContext` (`@react-router/cloudflare`): `context.cloudflare.ctx.waitUntil`; Vercel:
  `@vercel/functions`; Node: `setImmediate`. `createIndexNowContext(platform)` возвращает
  dispatcher под платформу.
- `action` хелпер `withIndexNow(actionFn)` открывает область и flush.
