# Configuration Schema

rascal is config-driven: the entire broker topology and behaviour are declared
in a user-supplied configuration object/JSON, validated against this schema. This
is the central user-facing contract of the library — a rebuild MUST reproduce its
shape.

### schema.json [MUST] <!-- id: file:lib/config/schema.json:schema.json -->

[lib/config/schema.json](https://github.com/cliftonc/rascal/blob/master/lib/config/schema.json#L1-L693)

JSON Schema **draft-07**. Top-level object with optional `vhosts`,
`publications`, `subscriptions`, `redeliveries`, `encryption`. `vhosts`,
`publications`, `subscriptions` use `patternProperties: ".*"` — keys are
user-chosen names mapping to the definitions below.

**Top-level properties**

| Property | Type / ref | Notes |
|----------|-----------|-------|
| vhosts | map<name, vhost> | Per-vhost topology + connections |
| publications | map<name, publication> | Global publications |
| subscriptions | map<name, subscription> | Global subscriptions |
| redeliveries | redeliveries | Redelivery counter registry |
| encryption | encryption | Named encryption profiles |

**Key definitions**

- **vhost**: `assert`/`check` (bool), `connectionStrategy` (`random`|`fixed`),
  single `connection` or array of `connections`, `concurrency` (int ≥1),
  `exchanges`/`queues`/`bindings` (each `oneOf` string-array shorthand or
  name→definition map), nested `publications`/`subscriptions`, and
  `publicationChannelPools` (`regularPool`/`confirmPool` → channelPool).
- **channelPool**: `autostart` (bool), `max`/`min` (int ≥0), and timeout/eviction
  ints — `evictionRunIntervalMillis`, `idleTimeoutMillis`, `rejectionDelayMillis`,
  `acquireTimeoutMillis`, `destroyTimeoutMillis`, `testOnBorrow`.
- **connection**: `anyOf` — a URL string, a `{ url, options, socketOptions,
  retry, management }` object, or a discrete `{ protocol (amqp|amqps), hostname,
  user, password, port, vhost, ... }` object.
- **connectOptions**: `frameMax`, `heartbeat`, `connection_timeout`,
  `channelMax` (all int ≥0).
- **socketOptions**: `timeout` (int ≥0).
- **management**: `anyOf` — `{ url, options.timeout }` or discrete
  `{ protocol (http|https), hostname, user, password, port, options.timeout }`.
  Used for the RabbitMQ management API (vhost assert/check).
- **exchange**: `name`, `assert`, `check`, `type`, `options` (`durable`,
  `internal`, `autoDelete`, `alternateExchange`, `expires`, `arguments`).
- **queue**: `name`, `assert`, `check`, `type`, `purge`, `options` (`exclusive`,
  `durable`, `autoDelete`, `messageTtl`, `expires`, `deadLetterExchange`,
  `deadLetterRoutingKey`, `maxLength`, `maxPriority`, `arguments`).
- **binding**: `name`, `source`, `destination`, `destinationType`
  (`queue`|`exchange`), `bindingKey`, `bindingKeys[]`, `qualifyBindingKeys`,
  `options`.
- **publication**: `name`, `vhost`, `exchange`, `queue`, `routingKey`, `confirm`,
  `timeout`, `encryption` (profile name), `options` (`expiration`, `userId`,
  `CC`/`BCC` (string or array), `priority`, `persistent`, `mandatory`,
  `contentType`, `contentEncoding`, `headers`, `replyTo`, `type`, `appId`).
- **subscription**: `name`, `vhost`, `queue`, `contentType`, `prefetch`,
  `channelPrefetch`, `retry`, `redeliveries` (`limit`/`timeout`/`counter`),
  `closeTimeout`, `encryption`, `promisifyAckOrNack`, `options` (`noAck`,
  `exclusive`, `priority`, `arguments`, `consumerTag`).
- **shovel**: `name`, `subscription`, `publication` (defined but not referenced
  from a top-level property in this schema).
- **retry**: `min`, `max` (int ≥0), `factor` (number ≥0), `strategy`
  (`exponential`|`linear`).
- **redeliveries**: `counters` (object — named counter configs).
- **encryption**: `key`, `algorithm`, `ivLength` (int ≥1).

Note: the schema is permissive (no top-level `required`/`additionalProperties:
false`); it documents/validates the recognised shape rather than rejecting
unknown keys.
