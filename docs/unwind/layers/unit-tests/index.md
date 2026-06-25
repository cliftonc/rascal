# Unit Tests

Rascal is a config-driven RabbitMQ (amqplib) wrapper. Its test layer mixes a handful of pure unit tests (backoff strategies, in-memory counter) with a much larger body of integration-style tests that connect to a real RabbitMQ broker and exercise the full config -> vhost -> publish/subscribe pipeline.

## Sections
- [Harness & Strategy](harness.md) - runner, docker dependency, shared patterns
- [Test Utilities](utilities.md) - `test/utils/amqputils.js`, `lib/config/tests.js`
- [Pure Unit Tests](pure-unit-tests.md) - backoff (exponential/linear), in-memory counter
- [Config Pipeline Tests](config-tests.md) - config, defaults, validation
- [Broker Tests](broker-tests.md) - broker.tests.js, brokerAsPromised.tests.js
- [Publication Tests](publication-tests.md) - publications.tests.js, publicationsAsPromised.tests.js
- [Subscription Tests](subscription-tests.md) - subscriptions.tests.js, subscriptionsAsPromised.tests.js
- [Vhost & Shovel Tests](vhost-shovel-tests.md) - vhost.tests.js, shovel.tests.js

## Runner

| Setting | Value |
|---------|-------|
| Runner | [zUnit](https://www.npmjs.com/package/zunit) v4 (`npm test` -> `zUnit`) |
| File pattern | `^[\w-]+.tests.js$` (zUnit config in package.json) |
| `pollute` | `true` (zUnit injects `describe`/`it` as globals) |
| Coverage | `nyc --report html --reporter lcov --reporter text-summary zUnit` |
| Assertions | Node core `assert` (no extra assertion lib) |
| Async helper | `async` (eachSeries, timesSeries, whilst, series) |
| Broker | `docker run rabbitmq:3.12.9-management-alpine` on ports 5672 (AMQP) + 15672 (management) |
| Management API | `http://guest:guest@localhost:15672` (used for vhost assert/check, connection killing) |

## Summary

| Suite | File | Style | Asserts |
|-------|------|-------|---------|
| Exponential Backoff | test/backoff/exponential.tests.js | pure unit | backoff growth, randomise range, cap, reset |
| Linear Backoff | test/backoff/linear.tests.js | pure unit | constant delay, range |
| In Memory Counter | test/caches/inMemory.tests.js | pure unit | incrementAndGet, LRU size limit |
| Config | test/config.tests.js | unit (no broker) | config parsing/expansion |
| Defaults | test/defaults.tests.js | unit (no broker) | default merging |
| Validation | test/validation.tests.js | unit (no broker) | config validation errors |
| Broker | test/broker.tests.js | integration | broker lifecycle (callback API) |
| Broker As Promised | test/brokerAsPromised.tests.js | integration | broker lifecycle (promise API) |
| Publications | test/publications.tests.js | integration | publish behavior (callback API) |
| Publications As Promised | test/publicationsAsPromised.tests.js | integration | publish behavior (promise API) |
| Subscriptions | test/subscriptions.tests.js | integration | subscribe + recovery (callback API) |
| Subscriptions As Promised | test/subscriptionsAsPromised.tests.js | integration | subscribe + recovery (promise API) |
| Shovel | test/shovel.tests.js | integration | shovel message transfer |
| Vhost | test/vhost.tests.js | integration | vhost topology + reconnection |
| amqputils | test/utils/amqputils.js | helper | broker introspection helpers |

_Analysis in progress..._
