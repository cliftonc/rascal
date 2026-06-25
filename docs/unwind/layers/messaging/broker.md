# Broker

The orchestrator. Holds the registries of vhosts, publications, subscriptions, and active subscriber sessions; exposes the public API (`publish`, `forward`, `subscribe`, lifecycle commands) and composes the init task chain.

## Broker.js [MUST] <!-- id: file:lib/amqp/Broker.js:Broker.js -->

[Broker.js](https://github.com/cliftonc/rascal/blob/master/lib/amqp/Broker.js#L1-L232) · `inherits(Broker, EventEmitter)`

### Factory: `Broker.create(config, [components], next)` [MUST]

Callback factory. Runs the preflight pipeline then constructs and initialises the broker.

```js
const preflight = async.compose(validate, configure); // configure THEN validate
create: function create(config, components, next) {
  if (arguments.length === 2) return create(config, {}, arguments[1]); // components optional
  const counters = _.defaults({}, components.counters, { stub, inMemory, inMemoryCluster });
  preflight(_.cloneDeep(config), (err, augmentedConfig) => {
    if (err) return next(err);
    new Broker(augmentedConfig, _.assign({}, components, { counters }))._init(next);
  });
}
```

- `preflight = async.compose(validate, configure)` — runs `configure` first, then `validate` (async.compose is right-to-left).
- Default counter components injected: `stub`, `inMemory` (`../counters/inMemory`), `inMemoryCluster` (`../counters/inMemoryCluster`.worker). User-supplied `components.counters` win via `_.defaults`.
- Config is deep-cloned before preflight.

### Init task chain [MUST]

```js
const init = async.compose(tasks.initShovels, tasks.initSubscriptions, tasks.initPublications, tasks.initCounters, tasks.initVhosts);
```

**Execution order (async.compose is right-to-left):** `initVhosts` → `initCounters` → `initPublications` → `initSubscriptions` → `initShovels`.

`_init` resets all registries, runs `init(config, { broker: self, components }, …)`, then sets a `keepActive` interval (`setInterval(_.noop, maxInterval)`, `maxInterval = 2147483647`) to keep the event loop alive, and yields `next(err, self)` via `setImmediate`.

### Lifecycle command chains [MUST]

Composed in the constructor:

```js
const nukeVhost = async.compose(tasks.deleteVhost, tasks.shutdownVhost, tasks.nukeVhost); // order: nukeVhost → shutdownVhost → deleteVhost
const purgeVhost = tasks.purgeVhost;
const forewarnVhost = tasks.forewarnVhost;
const shutdownVhost = tasks.shutdownVhost;
const bounceVhost = tasks.bounceVhost;
```

- **`shutdown(next)`** — for each vhost: `forewarnVhost`; then `unsubscribeAll`; then for each vhost: `shutdownVhost`; then `clearInterval(keepActive)`. Uses `async.eachSeries`.
- **`bounce(next)`** — `unsubscribeAll`, then `bounceVhost` per vhost (`async.eachSeries`).
- **`nuke(next)`** — `unsubscribeAll`, then `nukeVhost` per vhost; on success clears `vhosts`/`publications`/`subscriptions` registries and `clearInterval(keepActive)`.
- **`purge(next)`** — `purgeVhost` per vhost (`async.eachSeries`).

### Connection / registry methods [MUST]

```js
this.connect = (name, next) => { if (!vhosts[name]) return next(Error('Unknown vhost')); vhosts[name].connect(next); };
this.getConnections = () => Object.keys(vhosts).map(n => vhosts[n].getConnectionDetails());
this.getFullyQualifiedName = this.qualify = (vhost, name) => fqn.qualify(name, config.vhosts[vhost].namespace);
this._addVhost / _addPublication / _addSubscription   // registry mutators called by tasks
```

### Publish / forward [MUST]

```js
this.publish = (name, message, overrides, next) => {
  if (arguments.length === 3) return self.publish(name, message, {}, arguments[2]);   // overrides optional
  if (_.isString(overrides)) return self.publish(name, message, { routingKey: overrides }, next); // string => routingKey
  if (!publications[name]) return next(Error('Unknown publication'));
  publications[name].publish(message, overrides, next);
};
this.forward = (name, message, overrides, next) => { /* same arg-shuffling; checks config.publications[name]; publications[name].forward(...) */ };
```

`next` receives `(err, publicationSession)`.

### Subscribe / subscribeAll / unsubscribeAll [MUST]

```js
this.subscribe = (name, overrides, next) => {
  if (arguments.length === 2) return self.subscribe(name, {}, arguments[1]);
  if (!subscriptions[name]) return next(Error('Unknown subscription'));
  subscriptions[name].subscribe(overrides, (err, session) => {
    if (err) return next(err);
    sessions.push(session);   // track for unsubscribeAll
    next(null, session);
  });
};
this.subscribeAll = (filter, next) => { /* filter optional (default () => true); subscribes config.subscriptions matching filter via async.mapSeries */ };
this.unsubscribeAll = (next) => { async.each(sessions.slice(), (session, cb) => { sessions.shift(); session.cancel(cb); }, next); };
```

### Events

`Broker` extends `EventEmitter`. It re-emits all vhost events (see [vhost.md](vhost.md)) because `initVhosts` calls `forwardEvents(vhost, broker)`. It emits `error` directly when `unsubscribeAll` fails inside `shutdown`/`bounce`/`nuke`, and (via shovels) `error`/`cancelled`.

---

## BrokerAsPromised.js [MUST] <!-- id: file:lib/amqp/BrokerAsPromised.js:BrokerAsPromised.js -->

[BrokerAsPromised.js](https://github.com/cliftonc/rascal/blob/master/lib/amqp/BrokerAsPromised.js#L1-L87) · `inherits(BrokerAsPromised, EventEmitter)`

Promise wrapper around `Broker`.

### Factory: `BrokerAsPromised.create(...)` → `Promise<BrokerAsPromised>` [MUST]

```js
Broker.create(...args.concat((err, broker) => {
  if (err && !broker) return reject(err);
  broker.promises = true;                       // flips ackOrNack promisification path
  const brokerAsPromised = new BrokerAsPromised(broker);
  if (!err) return resolve(brokerAsPromised);
  err.broker = Symbol('broker-as-promised');    // partial-init: attach broker to error as non-enumerable Symbol prop
  Object.defineProperty(err, err.broker, { enumerable: false, value: brokerAsPromised });
  return reject(err);
}));
```

Sets `broker.promises = true` (this is read by `Subscription.getAckOrNack` to decide whether to return a promisified `ackOrNack`). On partial-initialisation error, the broker is still attached to the rejected error under a Symbol key.

### Method wrapping [MUST]

```js
forwardEvents(broker, this);  // re-emit all broker events on the promise wrapper
const methods = ['connect','nuke','purge','shutdown','bounce','publish','forward','unsubscribeAll'];
// each wrapped: returns new Promise, calls broker[method](...args, (err, result) => err ? reject : resolve(result))
```

`subscribe` resolves a `new SubscriberSessionAsPromised(session)`; `subscribeAll` maps each session through `SubscriberSessionAsPromised`. `config`, `getConnections`, `getFullyQualifiedName`/`qualify` are passed through directly.
