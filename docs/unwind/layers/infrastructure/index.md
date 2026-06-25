# Infrastructure

Build/dependency config, the configuration JSON Schema, CI/publish workflows,
git hooks, project docs, and the example apps for **rascal** — a config-driven
RabbitMQ / AMQP client built on [amqplib](https://www.npmjs.com/package/amqplib),
published to npm.

The library is plain CommonJS Node.js with no compile step: `main` is
`index.js` and the package ships source as-is. The single load-bearing contract
here is `lib/config/schema.json` (JSON Schema draft-07) which validates all user
broker configuration.

## Sections

| Section | Covers | Status |
|---------|--------|--------|
| [build-config.md](./build-config.md) | package.json, lockfile, lint/format config, npm packaging, husky hook, docs/license | done |
| [ci-cd.md](./ci-cd.md) | GitHub Actions CI + publish workflows, Jekyll docs site | done |
| [config-schema.md](./config-schema.md) | `lib/config/schema.json` — user config contract | done |
| [examples.md](./examples.md) | Reference example apps under `examples/` | done |

There is no dedicated `entrypoints.md` / `deployment.md`: rascal is a library,
not a deployable service. The "entrypoint" is `index.js` (a code-layer file, not
seeded here); deployment is npm publish, documented in [ci-cd.md](./ci-cd.md).

## Summary

- **Package**: CommonJS library, `main: index.js`, no build step. 10 runtime
  deps (async, lodash, generic-pool, lru-cache, uuid, xregexp, debug, etc.),
  `amqplib` is a peer dependency (`>=0.5.5`). `engines.node >=14`.
- **Config schema**: `lib/config/schema.json` (draft-07) defines vhosts,
  connections, exchanges, queues, bindings, publications, subscriptions, channel
  pools, retry, redeliveries, encryption — the key user-facing contract.
- **CI/Publish**: GitHub Actions. CI runs lint + zUnit tests against a RabbitMQ
  service container on Node 16/18/20/22/24. Publish runs on GitHub release and
  `npm publish`es with `NPM_AUTH_TOKEN`.
- **Examples**: 7 reference apps under `examples/` demonstrating the public API
  (simple, promises, default-exchange, busy-publisher, streams, advanced/cluster,
  mocha test). Tagged [DON'T] — reference only, not rebuilt.

## Item count

Seed lists 37 items (seed `count` field says 38; actual `items` array has 37).
All 37 are documented across the section files. See the [Excluded](#excluded)
note below — none are excluded; every seed id appears with its anchor.

## Excluded

None. Every seeded item is documented in one of the section files above.
