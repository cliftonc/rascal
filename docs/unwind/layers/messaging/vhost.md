# Vhost

Per-vhost connection and channel-pool manager. Owns the amqplib connection, two generic-pool channel pools (regular + confirm), the topology task chains, and the reconnection/backoff machinery.

## Vhost.js [MUST] <!-- id: file:lib/amqp/Vhost.js:Vhost.js -->

[Vhost.js](https://github.com/cliftonc/rascal/blob/master/lib/amqp/Vhost.js#L1-L517) · `inherits(Vhost, EventEmitter)`

Factory: `Vhost.create(config, components, next)` → `new Vhost(config, components).init(next)`.

### Topology task chains [MUST]

Composed in the constructor (`async.compose` runs right-to-left):

```js
const init = async.compose(tasks.closeChannels, tasks.applyBindings, tasks.purgeQueues, tasks.checkQueues, tasks.assertQueues, tasks.checkExchanges, tasks.assertExchanges, tasks.createChannels, tasks.createConnection, tasks.checkVhost, tasks.assertVhost);
const connect = async.compose(tasks.createConnection);
const purge   = async.compose(tasks.closeConnection, tasks.closeChannels, tasks.purgeQueues, tasks.createChannels, tasks.createConnection);
const nuke    = async.compose(tasks.closeConnection, tasks.closeChannels, tasks.deleteQueues, tasks.deleteExchanges, tasks.createChannels, tasks.createConnection);
```

**`init` execution order:** `assertVhost` → `checkVhost` → `createConnection` → `createChannels` → `assertExchanges` → `checkExchanges` → `assertQueues` → `checkQueues` → `purgeQueues` → `applyBindings` → `closeChannels`.

**`purge` order:** `createConnection` → `createChannels` → `purgeQueues` → `closeChannels` → `closeConnection`.

**`nuke` order:** `createConnection` → `createChannels` → `deleteExchanges` → `deleteQueues` → `closeChannels` → `closeConnection`.

These chains use a temporary set of plain channels (`ctx.channels`) for topology, separate from the long-lived publication channel pools.

### init(next) [MUST]

```js
this.init = function (next) {
  if (shuttingDown) return next();            // abort if shutting down
  pauseChannelAllocation();
  init(vhostConfig, { connectionIndex: self.connectionIndex, components }, (err, config, ctx) => {
    if (err) return next(err);
    connection = ctx.connection;
    self.connectionIndex = ctx.connectionIndex;
    connectionConfig = ctx.connectionConfig;
    timer = backoff(ctx.connectionConfig.retry);     // (re)build backoff timer from retry config
    attachDisconnectionHandlers(config);
    forwardRabbitMQConnectionEvents();               // blocked/unblocked
    ensureChannelPools();                            // lazily create regular + confirm pools
    resumeChannelAllocation();
    self.emit('connect');
    self.emit('vhost_initialised', self.getConnectionDetails());
    return next(null, self);
  });
};
```

### Lifecycle methods [MUST]

- **`forewarn(next)`** — `pauseChannelAllocation()`, sets `shuttingDown = true`, `channelCreator.resume()` (so in-flight channel requests drain/get rejected), `next()`.
- **`shutdown(next)`** — `clearTimeout(reconnectTimeout)`, pause allocation, `drainChannelPools` then `disconnect`.
- **`nuke(next)`** — pause allocation, `drainChannelPools`, then run `nuke` chain.
- **`purge(next)`** — run `purge` chain with `ctx.purge = true`.
- **`bounce(next)`** — `async.series([disconnect, init])`.
- **`connect(next)`** — run `connect` chain, yields `(err, ctx.connection)`.
- **`disconnect(next)`** — if no connection, return; else `removeAllListeners`, attach an error-swallowing handler, `connection.close`, clear `connection`.

### Channel acquisition API [MUST]

`channelCreator` is `async.queue(createChannel, 1)` (serialised, concurrency 1).

```js
this.getChannel        = (next) => channelCreator.push({ confirm: false }, next);
this.getConfirmChannel = (next) => channelCreator.push({ confirm: true }, next);

this.borrowChannel        = (next) => regularChannelPool ? regularChannelPool.borrow(next) : next(Error('must be initialised'));
this.returnChannel        = (channel) => regularChannelPool && regularChannelPool.release(channel);
this.destroyChannel       = (channel) => regularChannelPool && regularChannelPool.destroy(channel);
this.borrowConfirmChannel  / returnConfirmChannel / destroyConfirmChannel  // same against confirmChannelPool
this.isPaused = () => paused;
this.getConnectionDetails = () => ({ vhost: self.name, connectionUrl: connectionConfig.loggableUrl });
```

`getChannel`/`getConfirmChannel` create **fresh, unpooled** channels (used by subscriptions and republish). `borrow*`/`return*`/`destroy*` operate on the **pooled** channels (used by publications).

### Channel pools (generic-pool) [MUST]

`createChannelPool(options)` builds a `generic-pool` pool with a factory:

- **`create()`** — calls `createChannelWhenInitialised`; on the resulting channel binds a `_.once` `destroyChannel` to both `'error'` and `'close'` that sets `channel._rascal_closed = true` and `pool.destroy(channel)` if borrowed.
- **`destroy(channel)`** — if `_rascal_closed` resolve immediately; else `removeAllListeners`, close with a 1000ms `setTimeoutUnref` guard (because a dropped connection can make `channel.close` never call back — see code comment).
- **`validate(channel)`** — resolves `!channel._rascal_closed && connection && connection.connection === channel.connection` (rejects channels from a stale connection).

`deferRejection` delays generic-pool rejections by `options.pool.rejectionDelayMillis` via `setTimeoutUnref` to avoid a CPU-eating tight loop (references node-pool issue #197).

Pool wrapper exposes `{ stats, borrow, release, destroy, drain, pause, resume }`. `borrow` queues through a serialised `async.queue` (`poolQueue`, concurrency 1) that calls `pool.acquire()`. When `poolQueue.length() >= options.pool.max` it sets `busy = true` and emits `busy`; `checkReady` emits `ready` when the queue drains.

`ensureChannelPools()` creates two pools lazily (idempotent via `|| createChannelPool(...)`):
- `regularChannelPool` from `vhostConfig.publicationChannelPools.regularPool` (`confirm: false`)
- `confirmChannelPool` from `vhostConfig.publicationChannelPools.confirmPool` (`confirm: true`)

### createChannel / createChannelWhenInitialised [MUST]

`createChannelWhenInitialised(confirm, next)` — if connected, `createChannel` now; else defer until the next `vhost_initialised` event.

`createChannel(confirm, next)`:
- If `shuttingDown`, `next()` (no channel — callers treat falsy channel as "vhost shutting down").
- If no connection, `next(Error('must be initialised'))`.
- Wraps `next` in `_.once`; binds temporary `close`/`error` connection listeners that reject; calls `connection.createConfirmChannel(cb)` or `connection.createChannel(cb)`.
- Stamps `channel._rascal_id = uuid()`, copies `connection._rascal_id`, `setMaxListeners(0)`.
- Guards against amqplib double-callback (issue #388): if `invocations > 1`, closes the superfluous channel.

### Pause / resume / disconnect handling [MUST]

- `pauseChannelAllocation()` — pauses `channelCreator` + both pools, sets `paused = true`, emits `paused` (`{ vhost }`).
- `resumeChannelAllocation()` — resumes all, `paused = false`, emits `resumed` (`{ vhost }`).
- `forwardRabbitMQConnectionEvents()` — re-emits amqplib `blocked` (with reason) and `unblocked`.
- `attachDisconnectionHandlers(config)` — removes existing error listeners, binds a single `_.once` handler to both `error` and `close`.
- `makeDisconnectionHandler` — wraps in `setImmediate` (avoids amqplib accept-loop swallowing errors); routes close-with-error to `handleConnectionError`, plain close to `handleConnectionClose`.
- `handleConnectionError` — pause allocation, clear `connection`, emit `disconnect`, emit `error` (err + details), `retryConnection`.
- `handleConnectionClose` — pause allocation, clear `connection`, emit `disconnect`, emit `close` (details), `retryConnection`.

### Reconnection / backoff [MUST]

```js
function retryConnection(borked, config) {
  connectionConfig.retry && self.init((err) => {
    if (!err) return;
    const delay = timer.next();   // backoff timer (../backoff), built from connectionConfig.retry
    reconnectTimeout = setTimeoutUnref(handleConnectionError.bind(null, borked, config, err), delay);
  });
}
```

Reconnection only happens if `connectionConfig.retry` is truthy. On `init` failure it backs off via `timer.next()` and recursively re-triggers the error handler. `createConnection` itself round-robins across `config.connections` (see [tasks.md](tasks.md)).

### Events emitted by Vhost [MUST]

| Event | Payload | When |
|-------|---------|------|
| `connect` | — | after successful `init` |
| `vhost_initialised` | connection details | after `init`; also resolves deferred channel creation |
| `disconnect` | — | on connection error or close |
| `error` | `(err, connectionDetails)` | connection close-with-error |
| `close` | connection details | connection clean close |
| `blocked` | `(reason, connectionDetails)` | amqplib blocked |
| `unblocked` | connection details | amqplib unblocked |
| `paused` | `{ vhost }` | channel allocation paused |
| `resumed` | `{ vhost }` | channel allocation resumed |
| `busy` | pool stats | pool queue at max |
| `ready` | pool stats | pool queue drained after busy |

All of these are forwarded onto the `Broker` (`initVhosts` calls `forwardEvents(vhost, broker)`).
