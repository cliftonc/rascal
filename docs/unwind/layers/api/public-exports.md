# Public Exports

The package entry point. The object returned by `require('rascal')` is the complete public API contract.

Source: [index.js#L1-L24](https://github.com/cliftonc/rascal/blob/master/index.js#L1-L24)

```js
const _ = require('lodash');
const defaultConfig = require('./lib/config/defaults');
const testConfig = require('./lib/config/tests');
const Broker = require('./lib/amqp/Broker');
const BrokerAsPromised = require('./lib/amqp/BrokerAsPromised');
const counters = require('./lib/counters');

module.exports = (function () {
  return {
    Broker,
    BrokerAsPromised,
    createBroker: Broker.create,
    createBrokerAsPromised: BrokerAsPromised.create,
    defaultConfig,
    testConfig,
    withDefaultConfig(config) {
      return _.defaultsDeep({}, config, defaultConfig);
    },
    withTestConfig(config) {
      return _.defaultsDeep({}, config, testConfig);
    },
    counters,
  };
}());
```

The module export is an IIFE returning a static object literal. There are nine top-level keys, all enumerated below. Every key is part of the API contract and tagged `[MUST]`.

---

### Broker [MUST] <!-- id: file:index.js:index.js -->

The callback-style broker class, re-exported as-is from `./lib/amqp/Broker`.

Source: [index.js#L4](https://github.com/cliftonc/rascal/blob/master/index.js#L4-L4), [index.js#L10](https://github.com/cliftonc/rascal/blob/master/index.js#L10-L10)

- Exposed as the constructor/namespace `Broker`.
- Carries the static factory `Broker.create` (see `createBroker`).
- Instances inherit from `EventEmitter` ([Broker.js#L36](https://github.com/cliftonc/rascal/blob/master/lib/amqp/Broker.js#L36-L36)).
- Consumers normally construct a broker via the `createBroker` factory rather than `new Broker(...)` directly; the constructor expects an already-augmented (preflighted) config.

---

### BrokerAsPromised [MUST] <!-- id: export:index.js:BrokerAsPromised -->

The promise-style broker class, re-exported as-is from `./lib/amqp/BrokerAsPromised`.

Source: [index.js#L5](https://github.com/cliftonc/rascal/blob/master/index.js#L5-L5), [index.js#L11](https://github.com/cliftonc/rascal/blob/master/index.js#L11-L11)

- Wraps a callback `Broker`, converting its methods to promises ([BrokerAsPromised.js#L32-L40](https://github.com/cliftonc/rascal/blob/master/lib/amqp/BrokerAsPromised.js#L32-L40)).
- Promisified methods: `connect`, `nuke`, `purge`, `shutdown`, `bounce`, `publish`, `forward`, `unsubscribeAll` ([BrokerAsPromised.js#L33](https://github.com/cliftonc/rascal/blob/master/lib/amqp/BrokerAsPromised.js#L33-L33)).
- Inherits from `EventEmitter` and forwards events from the underlying broker via `forward-emitter` ([BrokerAsPromised.js#L30,L36](https://github.com/cliftonc/rascal/blob/master/lib/amqp/BrokerAsPromised.js#L30-L36)).
- Carries the static factory `BrokerAsPromised.create` (see `createBrokerAsPromised`).

---

### createBroker(config, [components], callback) [MUST] <!-- id: export:index.js:createBroker -->

Callback factory. Alias of `Broker.create`.

Source: [index.js#L12](https://github.com/cliftonc/rascal/blob/master/index.js#L12-L12) → [Broker.js#L20-L33](https://github.com/cliftonc/rascal/blob/master/lib/amqp/Broker.js#L20-L33)

```js
create: function create(config, components, next) {
  if (arguments.length === 2) return create(config, {}, arguments[1]);

  const counters = _.defaults({}, components.counters, {
    stub,
    inMemory,
    inMemoryCluster,
  });

  preflight(_.cloneDeep(config), (err, augmentedConfig) => {
    if (err) return next(err);
    new Broker(augmentedConfig, _.assign({}, components, { counters }))._init(next);
  });
},
```

Signature / contract:
- `config` (object, required) — the Rascal broker configuration.
- `components` (object, optional) — overridable dependencies, e.g. `counters`. When called with exactly **2 arguments**, the second argument is treated as the callback and `components` defaults to `{}`.
- `callback` / `next` (function, required) — node-style `(err, broker)`.
- Default `components.counters` is `{ stub, inMemory, inMemoryCluster }` (the worker variant of `inMemoryCluster`).
- The config is deep-cloned and passed through `preflight` (validation/configuration) before the `Broker` is constructed and `_init`-ialised.

---

### createBrokerAsPromised(config, [components]) [MUST] <!-- id: export:index.js:createBrokerAsPromised -->

Promise factory. Alias of `BrokerAsPromised.create`.

Source: [index.js#L13](https://github.com/cliftonc/rascal/blob/master/index.js#L13-L13) → [BrokerAsPromised.js#L9-L27](https://github.com/cliftonc/rascal/blob/master/lib/amqp/BrokerAsPromised.js#L9-L27)

```js
create() {
  const args = Array.prototype.slice.call(arguments);
  return new Promise((resolve, reject) => {
    Broker.create(
      ...args.concat((err, broker) => {
        if (err && !broker) return reject(err);
        broker.promises = true;
        const brokerAsPromised = new BrokerAsPromised(broker);
        if (!err) return resolve(brokerAsPromised);
        err.broker = Symbol('broker-as-promised');
        Object.defineProperty(err, err.broker, {
          enumerable: false,
          value: brokerAsPromised,
        });
        return reject(err);
      }),
    );
  });
},
```

Signature / contract:
- Same positional arguments as `createBroker` minus the callback: `(config)` or `(config, components)`.
- Returns a `Promise<BrokerAsPromised>`.
- Resolves with a `BrokerAsPromised` instance on success.
- On error **with** a broker available, rejects with the error and attaches the `BrokerAsPromised` instance to the error under a non-enumerable `Symbol` key stored at `err.broker` (so callers can still shut the partially-built broker down).
- On error **without** a broker, rejects with the raw error.

---

### defaultConfig [MUST] <!-- id: export:index.js:defaultConfig -->

The default Rascal configuration object, re-exported from `./lib/config/defaults`.

Source: [index.js#L2](https://github.com/cliftonc/rascal/blob/master/index.js#L2-L2), [index.js#L14](https://github.com/cliftonc/rascal/blob/master/index.js#L14-L14) → `lib/config/defaults.js`

- A plain configuration object containing sane default vhost/connection/publication/subscription settings, merged into user config by `withDefaultConfig`.

---

### testConfig [MUST] <!-- id: export:index.js:testConfig -->

The test configuration object, re-exported from `./lib/config/tests`.

Source: [index.js#L3](https://github.com/cliftonc/rascal/blob/master/index.js#L3-L3), [index.js#L15](https://github.com/cliftonc/rascal/blob/master/index.js#L15-L15) → `lib/config/tests.js`

- A configuration object tuned for tests (e.g. namespacing, auto-delete), merged into user config by `withTestConfig`.

---

### withDefaultConfig(config) [MUST] <!-- id: export:index.js:withDefaultConfig -->

Merges the supplied `config` over `defaultConfig` and returns a new object.

Source: [index.js#L16-L18](https://github.com/cliftonc/rascal/blob/master/index.js#L16-L18)

```js
withDefaultConfig(config) {
  return _.defaultsDeep({}, config, defaultConfig);
},
```

Contract:
- Returns `_.defaultsDeep({}, config, defaultConfig)` — a fresh object where `config` values take precedence and `defaultConfig` fills in any missing keys (deep merge).
- Pure: does not mutate `config` or `defaultConfig` (writes into a new `{}`).

---

### withTestConfig(config) [MUST] <!-- id: export:index.js:withTestConfig -->

Merges the supplied `config` over `testConfig` and returns a new object.

Source: [index.js#L19-L21](https://github.com/cliftonc/rascal/blob/master/index.js#L19-L21)

```js
withTestConfig(config) {
  return _.defaultsDeep({}, config, testConfig);
},
```

Contract:
- Returns `_.defaultsDeep({}, config, testConfig)` — a fresh object where `config` values take precedence and `testConfig` fills in missing keys (deep merge).
- Pure: does not mutate inputs.

---

### counters [MUST] <!-- id: export:index.js:counters -->

The counter implementations object, re-exported from `./lib/counters`.

Source: [index.js#L6](https://github.com/cliftonc/rascal/blob/master/index.js#L6-L6), [index.js#L22](https://github.com/cliftonc/rascal/blob/master/index.js#L22-L22) → [lib/counters/index.js#L1-L9](https://github.com/cliftonc/rascal/blob/master/lib/counters/index.js#L1-L9)

```js
const stub = require('./stub');
const inMemory = require('./inMemory');
const inMemoryCluster = require('./inMemoryCluster');

module.exports = {
  stub,
  inMemory,
  inMemoryCluster,
};
```

Contract — exposes three counter strategies (used for redelivery/retry counting):
- `counters.stub` — no-op counter.
- `counters.inMemory` — single-process in-memory counter.
- `counters.inMemoryCluster` — cluster-aware in-memory counter (master/worker variants; `Broker.create` uses the `.worker` form as its default).

These can be supplied to `createBroker`/`createBrokerAsPromised` via `components.counters` to override the defaults.
