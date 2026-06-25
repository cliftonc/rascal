# Entities

Domain entities are plain configuration objects. Their canonical shape and
default field values are defined in `baseline.js`, which is merged into every
user config via `_.defaultsDeep(rascalConfig, baseline)` (see
[configure.js:13](https://github.com/cliftonc/rascal/blob/master/lib/config/configure.js#L13)).
`baseline.js` is the *lowest-priority* layer: `tests.js`/`defaults.js` override
it, and explicit user config overrides everything.

The top-level config has four entity collections plus a `defaults` block:
`vhosts`, `publications`, `subscriptions`, `redeliveries.counters`, `shovels`
(and `encryption`). Each `vhost` in turn owns `exchanges`, `queues`, `bindings`,
and (pre-normalization) nested `publications`/`subscriptions`.

### baseline.js [MUST] <!-- id: file:lib/config/baseline.js:baseline.js -->

[baseline.js](https://github.com/cliftonc/rascal/blob/master/lib/config/baseline.js#L1-L110)

Defines the default field values (the entity shapes) for every domain entity.

```javascript
module.exports = {
  defaults: {
    vhosts: {
      concurrency: 1,
      publicationChannelPools: {
        regularPool: { autostart: false, max: 5, min: 1, evictionRunIntervalMillis: 10000, idleTimeoutMillis: 60000, rejectionDelayMillis: 1000, testOnBorrow: true, acquireTimeoutMillis: 15000, destroyTimeoutMillis: 1000 },
        confirmPool: { autostart: false, max: 5, min: 1, evictionRunIntervalMillis: 10000, idleTimeoutMillis: 60000, rejectionDelayMillis: 1000, testOnBorrow: true, acquireTimeoutMillis: 15000, destroyTimeoutMillis: 1000 },
      },
      connectionStrategy: 'random',
      connection: {
        slashes: true, protocol: 'amqp', hostname: 'localhost', user: 'guest', password: 'guest', port: '5672', options: {},
        retry: { min: 1000, max: 60000, factor: 2, strategy: 'exponential' },
        management: { slashes: true, protocol: 'http', port: 15672, options: {} },
      },
      exchanges: { assert: true, type: 'topic', options: {} },
      queues: { assert: true, options: { arguments: {} } },
      bindings: { destinationType: 'queue', bindingKey: '#', options: {} },
    },
    publications: { vhost: '/', confirm: true, timeout: 10000, options: { persistent: true, mandatory: true } },
    subscriptions: {
      vhost: '/', prefetch: 10,
      retry: { min: 1000, max: 60000, factor: 2, strategy: 'exponential' },
      redeliveries: { limit: 100, timeout: 1000, counter: 'stub' },
      options: {},
    },
    redeliveries: { counters: { stub: {}, inMemory: { size: 1000 } } },
    shovels: {},
  },
  publications: {},
  subscriptions: {},
  redeliveries: { counters: { stub: {} } },
};
```

The entity shapes derived from `baseline.js` follow. Each table lists the
default-supplied fields (the rebuild must reproduce these defaults).

#### Vhost entity [MUST]

| Field | Type | Default | Notes |
|-------|------|---------|-------|
| name | string | (key) | Set to the collection key during normalization ([configure.js:46](https://github.com/cliftonc/rascal/blob/master/lib/config/configure.js#L46)) |
| namespace | string \| boolean | `undefined` (baseline) / `true` (defaults.js) | `true` → replaced with a random UUID ([configure.js:54](https://github.com/cliftonc/rascal/blob/master/lib/config/configure.js#L54)) |
| concurrency | number | 1 | |
| connectionStrategy | enum | `random` | `random` \| `fixed` |
| publicationChannelPools | object | regularPool + confirmPool (see below) | Only `regularPool`/`confirmPool` keys allowed ([validate.js:54](https://github.com/cliftonc/rascal/blob/master/lib/config/validate.js#L54)) |
| connection / connections | object \| string \| array | localhost amqp guest:guest:5672 | Normalized into `connections` array ([configure.js:58-73](https://github.com/cliftonc/rascal/blob/master/lib/config/configure.js#L58-L73)) |
| exchanges | keyed collection | `{ '': {} }` (default nameless exchange) | |
| queues | keyed collection | `{}` | |
| bindings | keyed collection | `{}` | |

#### Connection entity [MUST]

| Field | Type | Default | Notes |
|-------|------|---------|-------|
| slashes | boolean | true | |
| protocol | string | `amqp` | |
| hostname | string | `localhost` | |
| user | string | `guest` | |
| password | string | `guest` | |
| port | string | `5672` | |
| options | object | `{}` | Merged with defaults.js `{ heartbeat: 10, connection_timeout: 10000, channelMax: 100 }`; overridden by `heartbeat: 50` in tests.js |
| socketOptions | object | `{ timeout: 10000 }` (defaults.js) | |
| retry | object | `{ min: 1000, max: 60000, factor: 2, strategy: 'exponential' }` | Reconnect backoff |
| management | object | `{ slashes: true, protocol: 'http', port: 15672, options: {} }` | RabbitMQ management API connection |
| url / loggableUrl / index / vhost | derived | computed | Computed during normalization ([configure.js:124-145](https://github.com/cliftonc/rascal/blob/master/lib/config/configure.js#L124-L145)); `loggableUrl` masks the password |

#### PublicationChannelPool entity [MUST]

Two pools per vhost: `regularPool` (non-confirm publications) and `confirmPool`
(confirm publications). Identical default shape:

```javascript
{ autostart: false, max: 5, min: 1, evictionRunIntervalMillis: 10000, idleTimeoutMillis: 60000, rejectionDelayMillis: 1000, testOnBorrow: true, acquireTimeoutMillis: 15000, destroyTimeoutMillis: 1000 }
```

#### Exchange entity [MUST]

| Field | Type | Default | Notes |
|-------|------|---------|-------|
| name | string | (key) | |
| fullyQualifiedName | string | `fqn.qualify(name, namespace)` | ([configure.js:282](https://github.com/cliftonc/rascal/blob/master/lib/config/configure.js#L282)) |
| assert | boolean | true | |
| type | string | `topic` | AMQP exchange type |
| options | object | `{}` | (durable:false added by defaults.js) |
| check | boolean | (optional) | Allowed attribute ([validate.js:68](https://github.com/cliftonc/rascal/blob/master/lib/config/validate.js#L68)) |

#### Queue entity [MUST]

| Field | Type | Default | Notes |
|-------|------|---------|-------|
| name | string | (key) | |
| fullyQualifiedName | string | `fqn.qualify(name, namespace, replyTo)` | replyTo used as uniqueness suffix ([configure.js:296](https://github.com/cliftonc/rascal/blob/master/lib/config/configure.js#L296)) |
| assert | boolean | true | |
| options.arguments | object | `{}` | `x-dead-letter-exchange` arg is FQN-qualified ([configure.js:351-356](https://github.com/cliftonc/rascal/blob/master/lib/config/configure.js#L351-L356)) |
| replyTo | string \| boolean | (optional) | `true` → replaced with random UUID ([configure.js:290](https://github.com/cliftonc/rascal/blob/master/lib/config/configure.js#L290)) |
| purge | boolean | (optional, true in defaults.js) | Allowed attributes also: `check`, `type` ([validate.js:76](https://github.com/cliftonc/rascal/blob/master/lib/config/validate.js#L76)) |

#### Binding entity [MUST]

| Field | Type | Default | Notes |
|-------|------|---------|-------|
| name | string | (key) | |
| source | string | (parsed from name) | Exchange name; required ([validate.js:85](https://github.com/cliftonc/rascal/blob/master/lib/config/validate.js#L85)) |
| destination | string | (parsed from name) | Required ([validate.js:86](https://github.com/cliftonc/rascal/blob/master/lib/config/validate.js#L86)) |
| destinationType | enum | `queue` | `queue` \| `exchange` ([validate.js:88](https://github.com/cliftonc/rascal/blob/master/lib/config/validate.js#L88)) |
| bindingKey | string | `#` | One binding per key after expansion ([configure.js:334-349](https://github.com/cliftonc/rascal/blob/master/lib/config/configure.js#L334-L349)) |
| bindingKeys | array | (optional) | Expanded into multiple bindings |
| qualifyBindingKeys | boolean | (optional) | If set, bindingKey is FQN-qualified ([configure.js:311-313](https://github.com/cliftonc/rascal/blob/master/lib/config/configure.js#L311-L313)) |
| options | object | `{}` | |

#### Publication entity [MUST]

| Field | Type | Default | Notes |
|-------|------|---------|-------|
| name | string | (key) | |
| vhost | string | `/` | Required ([validate.js:105](https://github.com/cliftonc/rascal/blob/master/lib/config/validate.js#L105)) |
| exchange \| queue | string | (one required, not both) | XOR invariant ([validate.js:106-107](https://github.com/cliftonc/rascal/blob/master/lib/config/validate.js#L106-L107)) |
| confirm | boolean | true | Selects confirmPool vs regularPool |
| timeout | number | 10000 | |
| options.persistent | boolean | true (baseline) / false (defaults.js) | |
| options.mandatory | boolean | true | |
| destination | string | derived | FQN of target exchange/queue ([configure.js:190](https://github.com/cliftonc/rascal/blob/master/lib/config/configure.js#L190)) |
| replyTo | string | (optional) | Resolved to reply-queue FQN ([configure.js:192-197](https://github.com/cliftonc/rascal/blob/master/lib/config/configure.js#L192-L197)) |
| encryption | string \| object | (optional) | String → resolved to named encryption profile ([configure.js:199-201](https://github.com/cliftonc/rascal/blob/master/lib/config/configure.js#L199-L201)) |
| routingKey | string | (optional) | Allowed attribute ([validate.js:104](https://github.com/cliftonc/rascal/blob/master/lib/config/validate.js#L104)) |
| autoCreated / deprecated | boolean | (optional) | `autoCreated` set for exchange/queue-derived publications |

#### Subscription entity [MUST]

| Field | Type | Default | Notes |
|-------|------|---------|-------|
| name | string | (key) | |
| vhost | string | `/` | Required ([validate.js:151](https://github.com/cliftonc/rascal/blob/master/lib/config/validate.js#L151)) |
| queue | string | (required) | ([validate.js:152](https://github.com/cliftonc/rascal/blob/master/lib/config/validate.js#L152)) |
| prefetch | number | 10 | |
| retry | object | `{ min: 1000, max: 60000, factor: 2, strategy: 'exponential' }` | |
| redeliveries | object | `{ limit: 100, timeout: 1000, counter: 'stub' }` | `counter` must reference a known counter ([validate.js:161](https://github.com/cliftonc/rascal/blob/master/lib/config/validate.js#L161)) |
| options | object | `{}` | |
| source | string | derived | FQN of source queue ([configure.js:237](https://github.com/cliftonc/rascal/blob/master/lib/config/configure.js#L237)) |
| encryption | object | derived from `config.encryption` | ([configure.js:238](https://github.com/cliftonc/rascal/blob/master/lib/config/configure.js#L238)) |
| closeTimeout | number | (500 in defaults.js) | Other allowed attrs: `contentType`, `channelPrefetch`, `recovery`, `workflow(s)`, `handler(s)`, `promisifyAckOrNack` ([validate.js:128-149](https://github.com/cliftonc/rascal/blob/master/lib/config/validate.js#L128-L149)) |

#### RedeliveryCounter entity [MUST]

| Field | Type | Default | Notes |
|-------|------|---------|-------|
| name | string | (key) | |
| type | string | (= name) | Selects default config path `defaults.redeliveries.counters.{type}` ([configure.js:269-275](https://github.com/cliftonc/rascal/blob/master/lib/config/configure.js#L269-L275)) |
| (stub) | object | `{}` | No-op counter |
| (inMemory).size | number | 1000 | Bounded in-memory counter |

#### Shovel entity [MUST]

| Field | Type | Default | Notes |
|-------|------|---------|-------|
| name | string | (key) | |
| subscription | string | (parsed / required) | Must reference known subscription ([validate.js:186](https://github.com/cliftonc/rascal/blob/master/lib/config/validate.js#L186)) |
| publication | string | (parsed / required) | Must reference known publication ([validate.js:187](https://github.com/cliftonc/rascal/blob/master/lib/config/validate.js#L187)) |

Name pattern `"<subscription> -> <publication>"` is parsed into the two fields
([configure.js:252-262](https://github.com/cliftonc/rascal/blob/master/lib/config/configure.js#L252-L262)).

#### EncryptionProfile entity [MUST]

| Field | Type | Default | Notes |
|-------|------|---------|-------|
| name | string | (key) | |
| key | string | (required) | ([validate.js:172](https://github.com/cliftonc/rascal/blob/master/lib/config/validate.js#L172)) |
| algorithm | string | (required) | ([validate.js:173](https://github.com/cliftonc/rascal/blob/master/lib/config/validate.js#L173)) |
| ivLength | number | (required) | ([validate.js:174](https://github.com/cliftonc/rascal/blob/master/lib/config/validate.js#L174)) |
