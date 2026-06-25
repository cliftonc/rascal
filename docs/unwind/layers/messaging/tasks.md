# Tasks

The `lib/amqp/tasks/` directory is a registry of small, composable async tasks that `Vhost` and `Broker` assemble into topology-assertion and lifecycle pipelines using `async.compose` (which runs right-to-left). Every task follows the same contract and threads a shared mutable `ctx`.

**Task contract:** each task is `_.curry((config, ctx, next) => …)` (the four trivial lifecycle delegates are plain functions), and signals completion with `next(err, config, ctx)`. Returning `config` and `ctx` lets the next composed task receive them as its leading arguments.

**`ctx` threading:** the shared context carries (as relevant) `connection`, `connectionConfig`, `connectionIndex`, `channels`, `vhosts`, `publications`, `counters`, `broker`, `components`, `vhost`, and the `purge` flag. Tasks mutate `ctx` in place and pass it forward.

**Concurrency:** topology tasks fan out across the pre-created channel array using `async.eachOfLimit(..., config.concurrency, ...)`, picking `ctx.channels[index % config.concurrency]` so work is spread round-robin over channels.

**Idempotency:** assert/check/delete/purge tasks are guarded by per-element config flags (`assert`, `check`, `purge`) and are safe to re-run; `assert*` calls map directly to amqplib's idempotent `assert*`/`bind*` operations.

Link format: `https://github.com/cliftonc/rascal/blob/master/{path}#L{start}-L{end}`

## index.js [MUST] <!-- id: file:lib/amqp/tasks/index.js:index.js -->

[index.js](https://github.com/cliftonc/rascal/blob/master/lib/amqp/tasks/index.js#L1-L25)

The barrel/registry. Re-exports every task module under its name, so `Vhost`/`Broker` can pull tasks by key and feed them to `async.compose`:

```js
exports.applyBindings = require('./applyBindings');
exports.assertExchanges = require('./assertExchanges');
exports.assertQueues = require('./assertQueues');
exports.assertVhost = require('./assertVhost');
exports.bounceVhost = require('./bounceVhost');
exports.checkExchanges = require('./checkExchanges');
exports.checkQueues = require('./checkQueues');
exports.checkVhost = require('./checkVhost');
exports.closeChannels = require('./closeChannels');
exports.closeConnection = require('./closeConnection');
exports.createChannels = require('./createChannels');
exports.createConnection = require('./createConnection');
exports.deleteExchanges = require('./deleteExchanges');
exports.deleteQueues = require('./deleteQueues');
exports.deleteVhost = require('./deleteVhost');
exports.initCounters = require('./initCounters');
exports.initPublications = require('./initPublications');
exports.initShovels = require('./initShovels');
exports.initSubscriptions = require('./initSubscriptions');
exports.initVhosts = require('./initVhosts');
exports.nukeVhost = require('./nukeVhost');
exports.purgeQueues = require('./purgeQueues');
exports.purgeVhost = require('./purgeVhost');
exports.forewarnVhost = require('./forewarnVhost');
exports.shutdownVhost = require('./shutdownVhost');
```

25 task modules. No logic of its own — it is purely the lookup surface consumers compose against.

## Connection / channel

### createConnection.js [MUST] <!-- id: file:lib/amqp/tasks/createConnection.js:createConnection.js -->

[createConnection.js](https://github.com/cliftonc/rascal/blob/master/lib/amqp/tasks/createConnection.js#L1-L75)

Establishes an amqplib connection with failover across the configured `config.connections` candidates.

- Uses `async.retry(candidates.length, ...)`: each attempt connects to `candidates[ctx.connectionIndex]`; on failure it advances `ctx.connectionIndex = (ctx.connectionIndex + 1) % candidates.length` so the next attempt tries the next broker (round-robin failover).
- On success sets `ctx.connection` and `ctx.connectionConfig`.
- The internal `connect(connectionConfig, cb)`:
  - calls `amqplib.connect(url, socketOptions, cb)`; wraps `cb` in `_.once` (guards the duplicate-callback amqplib bug, onebeyond/rascal#17).
  - rewrites the error message to include the `loggableUrl`.
  - stamps `connection._rascal_id = uuid()` for logging/validation.
  - **binds an `error` handler immediately** so errors during initialisation (e.g. a failing `checkExchanges`) are routed through the callback chain instead of bubbling to `uncaughtException`. (`Vhost` removes this temporary handler once init completes.)
  - closes any superfluous connection (`invocations > 1`, squaremo/amqp.node#388).
  - calls `connection.setMaxListeners(0)`.

### closeConnection.js [MUST] <!-- id: file:lib/amqp/tasks/closeConnection.js:closeConnection.js -->

[closeConnection.js](https://github.com/cliftonc/rascal/blob/master/lib/amqp/tasks/closeConnection.js#L1-L10)

Closes `ctx.connection`. Idempotent / safe: if `ctx.connection` is absent it returns immediately (`next(null, config, ctx)`); otherwise `ctx.connection.close(cb)` and propagates any close error.

### createChannels.js [MUST] <!-- id: file:lib/amqp/tasks/createChannels.js:createChannels.js -->

[createChannels.js](https://github.com/cliftonc/rascal/blob/master/lib/amqp/tasks/createChannels.js#L1-L19)

Pre-allocates the channel pool used by the topology tasks. `async.times(config.concurrency, (i, cb) => ctx.connection.createChannel(cb), ...)` creates `config.concurrency` channels and stores them on `ctx.channels`. On error, forwards `next(err, config, ctx)` without setting `ctx.channels`.

### closeChannels.js [MUST] <!-- id: file:lib/amqp/tasks/closeChannels.js:closeChannels.js -->

[closeChannels.js](https://github.com/cliftonc/rascal/blob/master/lib/amqp/tasks/closeChannels.js#L1-L18)

Closes every channel in `ctx.channels` (`async.each(ctx.channels, (c, cb) => c.close(cb), ...)`), then `delete ctx.channels` regardless of error and forwards. Pairs with `createChannels` to bracket the topology phase.

## Topology assert

### assertVhost.js [MUST] <!-- id: file:lib/amqp/tasks/assertVhost.js:assertVhost.js -->

[assertVhost.js](https://github.com/cliftonc/rascal/blob/master/lib/amqp/tasks/assertVhost.js#L1-L29)

Asserts the vhost exists via the RabbitMQ **management HTTP API** (not amqplib). Skips entirely unless `config.assert`. Builds `new Client(ctx.components.agent)` and, with the same `async.retry` failover pattern as `createConnection`, calls `client.assertVhost(config.name, connectionConfig.management, cb)` across candidates, advancing `ctx.connectionIndex` on failure and recording `ctx.connectionConfig` on success. Idempotent (the management assert is a PUT).

### assertExchanges.js [MUST] <!-- id: file:lib/amqp/tasks/assertExchanges.js:assertExchanges.js -->

[assertExchanges.js](https://github.com/cliftonc/rascal/blob/master/lib/amqp/tasks/assertExchanges.js#L1-L27)

Asserts every exchange in `config.exchanges` over the channel pool (`eachOfLimit(keys, concurrency, ...)`). Per exchange, `assertExchange`:
- skips unless `config.assert`;
- skips the default exchange (`config.fullyQualifiedName === ''`);
- calls `channel.assertExchange(fullyQualifiedName, type, options, cb)`.

Idempotent via amqplib `assertExchange`.

### assertQueues.js [MUST] <!-- id: file:lib/amqp/tasks/assertQueues.js:assertQueues.js -->

[assertQueues.js](https://github.com/cliftonc/rascal/blob/master/lib/amqp/tasks/assertQueues.js#L1-L23)

Asserts every queue in `config.queues` over the channel pool. Per queue, `assertQueue` skips unless `config.assert`, then `channel.assertQueue(fullyQualifiedName, options, next)`. Idempotent.

### applyBindings.js [MUST] <!-- id: file:lib/amqp/tasks/applyBindings.js:applyBindings.js -->

[applyBindings.js](https://github.com/cliftonc/rascal/blob/master/lib/amqp/tasks/applyBindings.js#L1-L45)

Applies all `config.bindings` over the channel pool. Dispatches on `binding.destinationType` to `bindQueue` or `bindExchange`:
- `bindQueue`: resolves `config.queues[binding.destination]` and `config.exchanges[binding.source]`; errors (`Unknown destination`/`Unknown source`) if either is missing; then `channel.bindQueue(dest.fqn, source.fqn, bindingKey, options, next)`.
- `bindExchange`: same with `config.exchanges[binding.destination]`; `channel.bindExchange(...)`.

Idempotent via amqplib bind operations. Validates the topology graph references before binding.

## Topology check

### checkVhost.js [SHOULD] <!-- id: file:lib/amqp/tasks/checkVhost.js:checkVhost.js -->

[checkVhost.js](https://github.com/cliftonc/rascal/blob/master/lib/amqp/tasks/checkVhost.js#L1-L29)

Verifies (does not create) the vhost via the management API. Skips unless `config.check`. Same `async.retry` failover + `Client` pattern as `assertVhost`, calling `client.checkVhost(config.name, connectionConfig.management, cb)`. Read-only; fails the chain if the vhost is absent.

### checkExchanges.js [SHOULD] <!-- id: file:lib/amqp/tasks/checkExchanges.js:checkExchanges.js -->

[checkExchanges.js](https://github.com/cliftonc/rascal/blob/master/lib/amqp/tasks/checkExchanges.js#L1-L23)

Verifies each `config.exchanges` entry over the channel pool. Per exchange, `checkExchange` skips unless `config.check`, then `channel.checkExchange(fullyQualifiedName, next)`. Read-only assertion that the exchange already exists.

### checkQueues.js [SHOULD] <!-- id: file:lib/amqp/tasks/checkQueues.js:checkQueues.js -->

[checkQueues.js](https://github.com/cliftonc/rascal/blob/master/lib/amqp/tasks/checkQueues.js#L1-L23)

Verifies each `config.queues` entry over the channel pool. Per queue, `checkQueue` skips unless `config.check`, then `channel.checkQueue(fullyQualifiedName, next)`. Read-only existence check.

## Topology delete / purge

### deleteVhost.js [SHOULD] <!-- id: file:lib/amqp/tasks/deleteVhost.js:deleteVhost.js -->

[deleteVhost.js](https://github.com/cliftonc/rascal/blob/master/lib/amqp/tasks/deleteVhost.js#L1-L30)

Deletes a vhost via the management API. Reads the vhost config as `config.vhosts[ctx.vhost.name]` and skips unless that vhost's `assert` is set. Uses the `async.retry` failover pattern but indexed by **`ctx.vhost.connectionIndex`** (not `ctx.connectionIndex`), calling `client.deleteVhost(vhostConfig.name, connectionConfig.management, cb)` across `vhostConfig.connections`. Destructive admin operation; part of `nuke`.

### deleteExchanges.js [SHOULD] <!-- id: file:lib/amqp/tasks/deleteExchanges.js:deleteExchanges.js -->

[deleteExchanges.js](https://github.com/cliftonc/rascal/blob/master/lib/amqp/tasks/deleteExchanges.js#L1-L23)

Deletes every `config.exchanges` entry over the channel pool. Per exchange, `deleteExchange` skips the default exchange (`fullyQualifiedName === ''`), then `channel.deleteExchange(fullyQualifiedName, {}, next)`. No `check`/`assert` guard — unconditionally deletes (other than the default exchange). Destructive.

### deleteQueues.js [SHOULD] <!-- id: file:lib/amqp/tasks/deleteQueues.js:deleteQueues.js -->

[deleteQueues.js](https://github.com/cliftonc/rascal/blob/master/lib/amqp/tasks/deleteQueues.js#L1-L22)

Deletes every `config.queues` entry over the channel pool via `channel.deleteQueue(fullyQualifiedName, {}, next)`. Unconditional / destructive.

### purgeQueues.js [SHOULD] <!-- id: file:lib/amqp/tasks/purgeQueues.js:purgeQueues.js -->

[purgeQueues.js](https://github.com/cliftonc/rascal/blob/master/lib/amqp/tasks/purgeQueues.js#L1-L23)

Purges messages from each `config.queues` entry over the channel pool. Per queue, `purgeQueue` runs only when `config.purge || ctx.purge` (so it honours both per-queue config and a context-wide purge request), then `channel.purgeQueue(fullyQualifiedName, next)`. Destructive to message content but not topology.

### purgeVhost.js [SHOULD] <!-- id: file:lib/amqp/tasks/purgeVhost.js:purgeVhost.js -->

[purgeVhost.js](https://github.com/cliftonc/rascal/blob/master/lib/amqp/tasks/purgeVhost.js#L1-L6)

Thin delegate (plain function, not curried): `ctx.vhost.purge(cb)` then `next(null, config, ctx)`, propagating any error. Drives the vhost-level purge which ultimately runs `purgeQueues` with the `ctx.purge` flag set.

## Init

### initVhosts.js [MUST] <!-- id: file:lib/amqp/tasks/initVhosts.js:initVhosts.js -->

[initVhosts.js](https://github.com/cliftonc/rascal/blob/master/lib/amqp/tasks/initVhosts.js#L1-L31)

Creates and registers all `Vhost` instances. Sets `ctx.vhosts = {}`, raises `ctx.broker`'s max-listeners to at least the vhost count, then `async.eachSeries` over `config.vhosts`:
- `Vhost.create(vhostConfig, ctx.components, cb)`;
- `vhost.setMaxListeners(0)`;
- `forwardEvents(vhost, ctx.broker)` (forward-emitter) so vhost events bubble to the broker;
- `ctx.broker._addVhost(vhost)` and `ctx.vhosts[vhostConfig.name] = vhost`.

### initPublications.js [MUST] <!-- id: file:lib/amqp/tasks/initPublications.js:initPublications.js -->

[initPublications.js](https://github.com/cliftonc/rascal/blob/master/lib/amqp/tasks/initPublications.js#L1-L24)

Creates and registers all publications. `async.eachSeries` over `config.publications`, calling `Publication.create(ctx.vhosts[config.vhost], publicationConfig, cb)` and `ctx.broker._addPublication(publication)`. Depends on `initVhosts` having populated `ctx.vhosts`.

### initSubscriptions.js [MUST] <!-- id: file:lib/amqp/tasks/initSubscriptions.js:initSubscriptions.js -->

[initSubscriptions.js](https://github.com/cliftonc/rascal/blob/master/lib/amqp/tasks/initSubscriptions.js#L1-L23)

Creates and registers all subscriptions. `async.eachSeries` over `config.subscriptions`, calling `Subscription.create(ctx.broker, ctx.vhosts[config.vhost], ctx.counters[config.redeliveries.counter], subscriptionConfig, cb)` and `ctx.broker._addSubscription(subscription)`. Depends on both `initVhosts` and `initCounters`. (Note: it adds the subscription before checking the create error, then calls `callback(err)`.)

### initCounters.js [MUST] <!-- id: file:lib/amqp/tasks/initCounters.js:initCounters.js -->

[initCounters.js](https://github.com/cliftonc/rascal/blob/master/lib/amqp/tasks/initCounters.js#L1-L25)

Builds the redelivery-counter instances used by subscriptions. Sets `ctx.counters = {}`, then `async.eachSeries` over `config.redeliveries.counters`. Per counter, `initCounter` looks up the factory `ctx.components.counters[config.type]`; errors `Unknown counter type: <type>` if absent; otherwise stores `ctx.components.counters[config.type](config)` under `ctx.counters[config.name]`. Must run before `initSubscriptions`.

### initSubscriptions / initShovels ordering

(Shovels reuse subscriptions and publications, so this runs after them.)

### initShovels.js [MUST] <!-- id: file:lib/amqp/tasks/initShovels.js:initShovels.js -->

[initShovels.js](https://github.com/cliftonc/rascal/blob/master/lib/amqp/tasks/initShovels.js#L1-L42)

Wires up shovels — message pumps that consume from one subscription and re-publish to a publication. `async.eachSeries` over `config.shovels`; per shovel, `initShovel`:
- `ctx.broker.subscribe(config.subscription, {}, cb)`;
- on each `message`, calls `ctx.broker.forward(config.publication, message, {}, ...)` and `ackOrNack()`s the source message on publication `success`;
- `subscription.on('error')` re-emits on the broker;
- `subscription.on('cancelled')` emits `cancelled` (falling back to `error` if unhandled).

Depends on subscriptions/publications already being registered.

## Lifecycle

The four lifecycle tasks are plain (non-curried) functions that delegate to the matching `Vhost` method and forward `next(null, config, ctx)`.

### forewarnVhost.js [MUST] <!-- id: file:lib/amqp/tasks/forewarnVhost.js:forewarnVhost.js -->

[forewarnVhost.js](https://github.com/cliftonc/rascal/blob/master/lib/amqp/tasks/forewarnVhost.js#L1-L6)

`ctx.vhost.forewarn(cb)` — signals an impending shutdown (e.g. begins draining) before `shutdownVhost`. Propagates error, else `next(null, config, ctx)`.

### shutdownVhost.js [MUST] <!-- id: file:lib/amqp/tasks/shutdownVhost.js:shutdownVhost.js -->

[shutdownVhost.js](https://github.com/cliftonc/rascal/blob/master/lib/amqp/tasks/shutdownVhost.js#L1-L6)

`ctx.vhost.shutdown(cb)` — gracefully closes the vhost's channels/connection.

### bounceVhost.js [SHOULD] <!-- id: file:lib/amqp/tasks/bounceVhost.js:bounceVhost.js -->

[bounceVhost.js](https://github.com/cliftonc/rascal/blob/master/lib/amqp/tasks/bounceVhost.js#L1-L6)

`ctx.vhost.bounce(cb)` — disconnects and reconnects the vhost (used to recover a vhost without a full teardown).

### nukeVhost.js [SHOULD] <!-- id: file:lib/amqp/tasks/nukeVhost.js:nukeVhost.js -->

[nukeVhost.js](https://github.com/cliftonc/rascal/blob/master/lib/amqp/tasks/nukeVhost.js#L1-L6)

`ctx.vhost.nuke(cb)` — destructive teardown (deletes topology / the vhost). Destructive admin operation, typically test-only.
