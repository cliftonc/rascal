# Value Objects

This layer has no embeddable classes. The "value objects" of the domain are
(1) the **fully-qualified name** (FQN) — a computed string identity derived from
a base name + namespace prefix + uniqueness suffix — and (2) the **default
config presets** that layer on top of `baseline.js` to produce the final entity
shapes.

## Fully-Qualified Naming

### fqn.js [MUST] <!-- id: file:lib/config/fqn.js:fqn.js -->

[fqn.js](https://github.com/cliftonc/rascal/blob/master/lib/config/fqn.js#L1-L20)

Computes the `fullyQualifiedName` for exchanges and queues (and FQN-qualified
binding keys / dead-letter-exchange args). The FQN is the actual AMQP resource
name; the short `name` is the config alias.

```javascript
function qualify(name, namespace, unique) {
  if (name === '') return name;            // nameless default exchange stays ''
  name = prefix(namespace, name);          // namespace:name
  name = suffix(unique || undefined, name);// name:unique
  return name;
}

function prefix(text, name, separator) {
  return text ? text + (separator || ':') + name : name;
}

function suffix(text, name, separator) {
  return text ? name + (separator || ':') + text : name;
}
```

Rules:
- Default separator is `:`.
- Empty `name` (`''`, the default nameless exchange) is returned unqualified.
- `namespace` falsy → no prefix; `unique` falsy → no suffix.
- Result form: `[namespace:]name[:unique]`. Example: namespace `app`, name
  `orders`, unique `abc` → `app:orders:abc`.
- `unique` is used for reply queues (the `replyTo` UUID) so each subscriber gets
  a distinct queue.

## Default Config Presets

The final config is built by deep-merging three layers, lowest priority last:
explicit user config → preset (`tests.js` or `defaults.js`) → `baseline.js`.

### defaults.js [MUST] <!-- id: file:lib/config/defaults.js:defaults.js -->

[defaults.js](https://github.com/cliftonc/rascal/blob/master/lib/config/defaults.js#L1-L21)

The **production** default preset. Sole exported object, deep-merged onto user
config (above `baseline.js`).

```javascript
module.exports = {
  defaults: {
    vhosts: {
      connection: {
        options: { heartbeat: 10, connection_timeout: 10000, channelMax: 100 },
        socketOptions: { timeout: 10000 },
        management: { options: { timeout: 1000 } },
      },
    },
  },
};
```

Adds connection-level defaults that `baseline.js` leaves empty: AMQP heartbeat
(10s), connection timeout (10s), channel max (100), socket timeout (10s),
management API timeout (1s).

### tests.js [SHOULD] <!-- id: file:lib/config/tests.js:tests.js -->

[tests.js](https://github.com/cliftonc/rascal/blob/master/lib/config/tests.js#L1-L44)

The **test** default preset. Deep-merges its own overrides *onto* `defaults.js`
(`_.defaultsDeep(testOverrides, defaultConfig)`), so it is a superset of the
production defaults tuned for fast, ephemeral test topologies. Tagged [SHOULD]
because it is a test-only convenience, not part of the production contract.

```javascript
const _ = require('lodash').runInContext();
const defaultConfig = require('./defaults');

module.exports = _.defaultsDeep(
  {
    defaults: {
      vhosts: {
        connection: { options: { heartbeat: 50 } },  // overrides defaults.js heartbeat: 10
        namespace: true,                              // unique vhost namespace per run (UUID)
        exchanges: { options: { durable: false } },   // ephemeral exchanges
        queues: { purge: true, options: { durable: false } }, // ephemeral, auto-purged queues
      },
      publications: { options: { persistent: false } },  // non-persistent messages
      subscriptions: { closeTimeout: 500 },
    },
    redeliveries: { counters: { inMemory: { size: 1000 } } },
  },
  defaultConfig,
);
```

Key test-only behaviors:
- `namespace: true` → every vhost gets a random-UUID namespace, isolating
  parallel test runs (resolved to a UUID in [configure.js:54](https://github.com/cliftonc/rascal/blob/master/lib/config/configure.js#L54)).
- `durable: false` on exchanges/queues and `persistent: false` on publications →
  nothing survives broker restart.
- `queues.purge: true` → queues are purged on startup.
- `heartbeat: 50` overrides the production `heartbeat: 10`.
