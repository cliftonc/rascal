# Harness & Strategy

## Runner

Tests run under [zUnit](https://www.npmjs.com/package/zunit) v4. Configuration lives in `package.json`:

[package.json](https://github.com/cliftonc/rascal/blob/master/package.json#L37-L68)

```json
"scripts": {
  "test": "zUnit",
  "coverage": "nyc --report html --reporter lcov --reporter text-summary zUnit",
  "docker": "docker run -d --name rascal-rabbitmq -p 5672:5672 -p 15672:15672 rabbitmq:3.12.9-management-alpine"
},
"zUnit": {
  "pollute": true,
  "pattern": "^[\\w-]+.tests.js$"
}
```

- `pattern` `^[\w-]+.tests.js$` selects test files (matches `broker.tests.js` etc., recursively under `test/`).
- `pollute: true` makes zUnit inject `describe` / `it` as globals (no per-file `require` of the runner).
- zUnit's `it` callback signature is `(test, done)` for async tests, or a function returning a Promise. Synchronous tests use a no-arg function. Per-test and per-suite options are passed as a trailing object, e.g. `it('name', fn, { timeout: 60000 })` and `}, { timeout: 6000 })` on the `describe`.

## Broker dependency

Most suites are integration tests against a live RabbitMQ. The expected broker:

- Image `rabbitmq:3.12.9-management-alpine` (management plugin required).
- AMQP on `localhost:5672`, management HTTP API on `localhost:15672`.
- Default credentials `guest:guest`.

The management API is used directly by tests and helpers to assert/check/delete vhosts and to forcibly kill connections (simulating broker-side disconnects for recovery tests).

Pure unit suites that need NO broker: `test/backoff/*`, `test/caches/inMemory.tests.js`, `test/config.tests.js`, `test/defaults.tests.js`, `test/validation.tests.js`.

## Shared test config

Integration suites merge their per-test config over a shared base via `lodash` `defaultsDeep`:

[lib/config/tests.js](https://github.com/cliftonc/rascal/blob/master/lib/config/tests.js)

```js
const _ = require('lodash').runInContext();
const defaultConfig = require('./defaults');

module.exports = _.defaultsDeep({
  defaults: {
    vhosts: {
      connection: { options: { heartbeat: 50 } },
      namespace: true,
      exchanges: { options: { durable: false } },
      queues: { purge: true, options: { durable: false } },
    },
    publications: { options: { persistent: false } },
    subscriptions: { closeTimeout: 500 },
  },
  redeliveries: { counters: { inMemory: { size: 1000 } } },
}, defaultConfig);
```

This base makes test topology ephemeral: non-durable exchanges/queues, purge-on-create, short heartbeat, and `namespace: true` so each suite/test namespaces its objects with a per-test `uuid()` to avoid cross-test collisions.

## Common per-suite pattern (integration suites)

```js
const Broker = require('..').Broker;            // or BrokerAsPromised
const AmqpUtils = require('./utils/amqputils');
const testConfig = require('../lib/config/tests');

describe('Suite', () => {
  let broker, amqputils, namespace, vhosts;

  beforeEach((test, done) => {
    namespace = uuid();                         // unique namespace per test
    vhosts = { '/': { namespace, exchanges: {...}, queues: {...}, bindings: {...} } };
    amqplib.connect((err, connection) => {      // raw amqplib connection for assertions
      amqputils = AmqpUtils.init(connection);
      done();
    });
  });

  afterEach((test, done) => {
    amqputils.disconnect(() => {
      if (broker) return broker.nuke(done);     // tear down topology + connections
      done();
    });
  });

  function createBroker(config, next) {
    config = _.defaultsDeep(config, testConfig);
    Broker.create(config, (err, _broker) => { broker = _broker; next(err, broker); });
  }
}, { timeout: 2000 });                          // suite-level timeout
```

Key conventions:
- Each suite owns a private `createBroker` that deep-merges the per-test config over `testConfig` and captures the broker for `afterEach` teardown (`broker.nuke`).
- A separate raw `amqplib` connection (wrapped by `amqputils`) is used to introspect the broker independently of rascal, asserting messages actually landed / exchanges & queues exist.
- Callback suites use the `(test, done)` signature; `*AsPromised` suites either return a Promise or, for event-driven assertions, fall back to `(test, done)`.

## Dual callback + promise APIs

Every behavioral area is tested twice: a callback suite (`broker.tests.js`, `publications.tests.js`, `subscriptions.tests.js`) and an `AsPromised` mirror (`brokerAsPromised.tests.js`, etc.) that exercises the same behavior through `BrokerAsPromised` and `.then()/.catch()`. The promise suites assert constructor names `SubscriberSessionAsPromised` vs the callback `SubscriberSession`.
