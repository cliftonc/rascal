# Examples

Reference apps under `examples/` demonstrating rascal's public API. They are
**not core library code** and are excluded from the npm tarball and from linting.
All are tagged **[DON'T]** — they are documentation/sample assets, reproduced
only if the rebuild wants equivalent demos. Each example's `config.json` declares
`"$schema": "../../lib/config/schema.json"` and follows the
[config schema](./config-schema.md). All require the library via `require('../..')`.

## advanced/

User-registration simulation: cluster master/worker, multi-connection vhost,
content schemas, recoverable vs dead-letter error handling, redelivery limits.

### advanced/index.js [DON'T] <!-- id: file:examples/advanced/index.js:index.js -->
[source](https://github.com/cliftonc/rascal/blob/master/examples/advanced/index.js#L1-L101) —
Creates a broker with `withDefaultConfig`, dynamically loads a handler per
subscription, wires `message`/`invalid_content`/`redeliveries_exceeded`/`cancel`/
`error` events, publishes simulated user events every 1s, and handles
SIGINT/SIGTERM/unhandledRejection/uncaughtException with `broker.shutdown`.

### advanced/cluster.js [DON'T] <!-- id: file:examples/advanced/cluster.js:cluster.js -->
[source](https://github.com/cliftonc/rascal/blob/master/examples/advanced/cluster.js#L1-L12) —
Node `cluster` bootstrap: master starts `Rascal.counters.inMemoryCluster.master()`
and forks (re-forking on exit); workers run `./index`.

### advanced/config.js [DON'T] <!-- id: file:examples/advanced/config.js:config.js -->
[source](https://github.com/cliftonc/rascal/blob/master/examples/advanced/config.js#L1-L202) —
JS config module (not JSON) exporting `{ rascal: {...} }`: a `customer-vhost`
with `assert`, a multi-entry `connections` array (cluster failover), exchanges,
queues, bindings, publications/subscriptions and recovery routing.

### advanced/handlers/saveUser.js [DON'T] <!-- id: file:examples/advanced/handlers/saveUser.js:saveUser.js -->
[source](https://github.com/cliftonc/rascal/blob/master/examples/advanced/handlers/saveUser.js#L1-L35) —
Handler factory `(broker) => (user, cb)`; randomly emits recoverable/unrecoverable
errors, else publishes `save_user_succeeded`.

### advanced/handlers/deleteUser.js [DON'T] <!-- id: file:examples/advanced/handlers/deleteUser.js:deleteUser.js -->
[source](https://github.com/cliftonc/rascal/blob/master/examples/advanced/handlers/deleteUser.js#L1-L19) —
Handler factory publishing `delete_user_succeeded`; throws on a crash flag to
demonstrate uncaught-exception recovery.

## simple/

Minimal callback-API publish/subscribe loop.

### simple/index.js [DON'T] <!-- id: file:examples/simple/index.js:index.js -->
[source](https://github.com/cliftonc/rascal/blob/master/examples/simple/index.js#L1-L24) —
`Broker.create` + `withDefaultConfig`, subscribe to `demo_sub`, publish to
`demo_pub` every 1s.

### simple/config.json [DON'T] <!-- id: file:examples/simple/config.json:config.json -->
[source](https://github.com/cliftonc/rascal/blob/master/examples/simple/config.json#L1-L37) —
Default vhost `/` with a confirmPool, demo exchange/queue/binding, publication
and subscription.

## promises/

Same as simple but using the `BrokerAsPromised` / async-await API.

### promises/index.js [DON'T] <!-- id: file:examples/promises/index.js:index.js -->
[source](https://github.com/cliftonc/rascal/blob/master/examples/promises/index.js#L1-L32) —
`await BrokerAsPromised.create`, `await broker.subscribe`, publish in an async
interval.

### promises/config.json [DON'T] <!-- id: file:examples/promises/config.json:config.json -->
[source](https://github.com/cliftonc/rascal/blob/master/examples/promises/config.json#L1-L29) —
Default vhost with `heartbeat: 5`, demo exchange/queue/binding, pub/sub.

## default-exchange/

Publishing to the AMQP default ("") exchange (routes by queue name).

### default-exchange/index.js [DON'T] <!-- id: file:examples/default-exchange/index.js:index.js -->
[source](https://github.com/cliftonc/rascal/blob/master/examples/default-exchange/index.js#L1-L24) —
Publishes with an explicit routing key (`demo_q`) through the default exchange.

### default-exchange/config.json [DON'T] <!-- id: file:examples/default-exchange/config.json:config.json -->
[source](https://github.com/cliftonc/rascal/blob/master/examples/default-exchange/config.json#L1-L24) —
Uses `exchanges: [""]` and a publication with `exchange: ""`.

## busy-publisher/

Demonstrates flow control (`busy`/`ready` events) under a high-rate stream.

### busy-publisher/index.js [DON'T] <!-- id: file:examples/busy-publisher/index.js:index.js -->
[source](https://github.com/cliftonc/rascal/blob/master/examples/busy-publisher/index.js#L1-L32) —
Pipes a `random-readable` stream into `broker.publish`, pausing/resuming the
stream on `busy`/`ready`.

### busy-publisher/config.json [DON'T] <!-- id: file:examples/busy-publisher/config.json:config.json -->
[source](https://github.com/cliftonc/rascal/blob/master/examples/busy-publisher/config.json#L1-L34) —
Default vhost with a sized `regularPool` (max/min 10) to trigger backpressure.

### busy-publisher/package.json [DON'T] <!-- id: file:examples/busy-publisher/package.json:package.json -->
[source](https://github.com/cliftonc/rascal/blob/master/examples/busy-publisher/package.json#L1-L14) —
Standalone manifest; sole dependency `random-readable ^1.0.1`.

### busy-publisher/package-lock.json [DON'T] <!-- id: file:examples/busy-publisher/package-lock.json:package-lock.json -->
[source](https://github.com/cliftonc/rascal/blob/master/examples/busy-publisher/package-lock.json#L1-L13) —
Lockfile (lockfileVersion 1) pinning `random-readable@1.0.1`.

## streams/

RabbitMQ stream queue (`x-queue-type: stream`) publish + offset-based subscribe.

### streams/publisher.js [DON'T] <!-- id: file:examples/streams/publisher.js:publisher.js -->
[source](https://github.com/cliftonc/rascal/blob/master/examples/streams/publisher.js#L1-L38) —
Publishes a bounded random stream (max from `argv[2]`) with busy/ready flow
control, shutting down on stream close.

### streams/subscriber.js [DON'T] <!-- id: file:examples/streams/subscriber.js:subscriber.js -->
[source](https://github.com/cliftonc/rascal/blob/master/examples/streams/subscriber.js#L1-L25) —
Subscribes with an `x-stream-offset` override (from `argv[2]`, default `first`).

### streams/publisher-config.json [DON'T] <!-- id: file:examples/streams/publisher-config.json:publisher-config.json -->
[source](https://github.com/cliftonc/rascal/blob/master/examples/streams/publisher-config.json#L1-L37) —
Declares a `demo_stream` queue (`x-queue-type: stream`, `x-max-length-bytes`)
and a non-confirm publication.

### streams/subscriber-config.json [DON'T] <!-- id: file:examples/streams/subscriber-config.json:subscriber-config.json -->
[source](https://github.com/cliftonc/rascal/blob/master/examples/streams/subscriber-config.json#L1-L28) —
Same stream queue plus a `demo_sub` subscription with `prefetch: 250`.

### streams/package.json [DON'T] <!-- id: file:examples/streams/package.json:package.json -->
[source](https://github.com/cliftonc/rascal/blob/master/examples/streams/package.json#L1-L14) —
Standalone manifest; sole dependency `random-readable ^1.0.1`.

### streams/package-lock.json [DON'T] <!-- id: file:examples/streams/package-lock.json:package-lock.json -->
[source](https://github.com/cliftonc/rascal/blob/master/examples/streams/package-lock.json#L1-L13) —
Lockfile pinning `random-readable@1.0.1`.

## mocha/

TDD pattern using `withTestConfig`, `broker.purge`/`nuke`.

### mocha/test.js [DON'T] <!-- id: file:examples/mocha/test.js:test.js -->
[source](https://github.com/cliftonc/rascal/blob/master/examples/mocha/test.js#L1-L40) —
Mocha-style test: creates a broker with `withTestConfig`, purges before each
test, nukes after, and asserts a published message is received.

### mocha/config.json [DON'T] <!-- id: file:examples/mocha/config.json:config.json -->
[source](https://github.com/cliftonc/rascal/blob/master/examples/mocha/config.json#L1-L21) —
Default vhost with demo exchange/queue/binding and pub/sub for the test.
