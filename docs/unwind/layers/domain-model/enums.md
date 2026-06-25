# Enums

The domain has no enum types in the language sense. The closed value sets below
are enforced by validation logic or referenced by configure logic. Each is a
business constraint the rebuild must reproduce.

### Connection strategy enum [MUST]

Selects how a vhost picks among its connections.

```javascript
type ConnectionStrategy = 'random' | 'fixed'   // default 'random'
```

| Value | Description |
|-------|-------------|
| random | Connection index is a stable random number keyed by `hostname:port` ([configure.js:142-145](https://github.com/cliftonc/rascal/blob/master/lib/config/configure.js#L142-L145)) |
| fixed | Connection index is the array position (`index`) ([configure.js:138-139](https://github.com/cliftonc/rascal/blob/master/lib/config/configure.js#L138-L139)) |

Validated against `[undefined, 'random', 'fixed']`; anything else throws "unknown
connection strategy" ([validate.js:38-40](https://github.com/cliftonc/rascal/blob/master/lib/config/validate.js#L38-L40)).

### Retry strategy enum [MUST]

Reconnect / subscription-recovery backoff strategy (default value, not validated
in this layer).

```javascript
type RetryStrategy = 'exponential'   // baseline default
```

| Value | Description |
|-------|-------------|
| exponential | Backoff `min..max` with `factor` (baseline: min 1000ms, max 60000ms, factor 2) — connection retry ([baseline.js:38-43](https://github.com/cliftonc/rascal/blob/master/lib/config/baseline.js#L38-L43)) and subscription retry ([baseline.js:80-85](https://github.com/cliftonc/rascal/blob/master/lib/config/baseline.js#L80-L85)) |

### Binding destination type enum [MUST]

Whether a binding targets a queue or an exchange.

```javascript
type DestinationType = 'queue' | 'exchange'   // default 'queue'
```

| Value | Description |
|-------|-------------|
| queue | Destination resolved against `vhost.queues` ([validate.js:93-95](https://github.com/cliftonc/rascal/blob/master/lib/config/validate.js#L93-L95)) |
| exchange | Destination resolved against `vhost.exchanges` ([validate.js:96](https://github.com/cliftonc/rascal/blob/master/lib/config/validate.js#L96)) |

Validated by `['queue', 'exchange'].indexOf(...) < 0` → "invalid destination
type" ([validate.js:88](https://github.com/cliftonc/rascal/blob/master/lib/config/validate.js#L88)).

### Exchange type enum [MUST]

AMQP exchange type. Default `topic`; not constrained in this layer (passed to
amqplib). Standard AMQP values: `topic`, `direct`, `fanout`, `headers`.
Default: [baseline.js:53](https://github.com/cliftonc/rascal/blob/master/lib/config/baseline.js#L53).

### Redelivery counter type enum [MUST]

Built-in redelivery-counter implementations (extensible — `type` defaults to the
counter's name and resolves defaults at `defaults.redeliveries.counters.{type}`,
[configure.js:271-273](https://github.com/cliftonc/rascal/blob/master/lib/config/configure.js#L271-L273)).

```javascript
type CounterType = 'stub' | 'inMemory'
```

| Value | Description |
|-------|-------------|
| stub | No-op counter, default for subscriptions (`redeliveries.counter: 'stub'`); always present in baseline ([baseline.js:95](https://github.com/cliftonc/rascal/blob/master/lib/config/baseline.js#L95)) |
| inMemory | Bounded in-memory counter, default `size: 1000` ([baseline.js:96-98](https://github.com/cliftonc/rascal/blob/master/lib/config/baseline.js#L96-L98)) |

### Publication channel pool name enum [MUST]

Closed set of channel-pool names per vhost.

```javascript
type PoolName = 'regularPool' | 'confirmPool'
```

| Value | Selected when | 
|-------|---------------|
| regularPool | publication `confirm` is false |
| confirmPool | publication `confirm` is true (baseline default) |

Validated: only these two keys allowed ([validate.js:54](https://github.com/cliftonc/rascal/blob/master/lib/config/validate.js#L54)).

### Protocol enums [MUST]

| Field | Default | Notes |
|-------|---------|-------|
| connection.protocol | `amqp` | `amqps` for TLS ([baseline.js:32](https://github.com/cliftonc/rascal/blob/master/lib/config/baseline.js#L32)) |
| management.protocol | `http` | `https` for TLS ([baseline.js:46](https://github.com/cliftonc/rascal/blob/master/lib/config/baseline.js#L46)) |
