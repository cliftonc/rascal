# Subscription

Inbound message abstraction. Lazily subscribes (only once a `message` listener is attached), decodes/decrypts content, tracks redeliveries, hands the app an `ackOrNack` callback, and resubscribes with backoff on channel failure.

## Subscription.js [MUST] <!-- id: file:lib/amqp/Subscription.js:Subscription.js -->

[Subscription.js](https://github.com/cliftonc/rascal/blob/master/lib/amqp/Subscription.js#L1-L335)

Factory: `Subscription.create(broker, vhost, counter, config, next)` → `new Subscription(broker, vhost, config, counter).init(next)`.

Constructor state: `timer = backoff(subscriptionConfig.retry)`, `subscriberError = new SubscriberError(broker, vhost)`, `sequentialChannelOperations = async.queue((task, next) => task(next), 1)` (serialised channel ops).

### subscribe(overrides, next) + lazy subscription [MUST]

```js
this.subscribe = function (overrides, next) {
  const config = _.defaultsDeep(overrides, subscriptionConfig);
  const session = new SubscriberSession(sequentialChannelOperations, config);
  subscribeLater(session, config);
  return next(null, session);
};

function subscribeLater(session, config) {
  session.on('newListener', (event) => {
    if (event !== 'message') return;        // only subscribe to the broker once the app listens for 'message'
    subscribeNow(session, config, (err) => {
      if (err) return session.emit('error', err);
      session.emit('subscribed');
    });
  });
}
```

**Lazy subscription contract:** no broker consume happens until the caller attaches a `message` listener. The session is returned synchronously.

### subscribeNow [MUST]

Pushed onto `sequentialChannelOperations` (serialised):

```js
if (session.isCancelled()) return done();
vhost.getChannel((err, channel) => {                 // fresh unpooled channel
  if (err) return done(err);
  if (!channel) return done();                       // vhost shutting down
  _configureQos(config, channel, (err) => {
    const removeDisconnectionHandlers = attachDisconnectionHandlers(channel, session, config);
    const onMessage = _onMessage.bind(null, session, config, removeDisconnectionHandlers);
    channel.consume(config.source, onMessage, config.options, (err, response) => {
      if (err) { removeDisconnectionHandlers(); return done(err); }
      session._open(channel, response.consumerTag, (err) => {
        if (err) return done(err);
        timer.reset();                                // reset backoff on successful subscribe
        done();
      });
    });
  });
});
```

`_configureQos`: if `config.prefetch` → `channel.prefetch(prefetch, false)` (consumer-level); if `config.channelPrefetch` → `channel.prefetch(channelPrefetch, true)` (channel-level). Run in `async.series`.

### _onMessage pipeline [MUST]

```js
if (!message) return handleConsumerCancel(...);                 // null => broker cancelled consumer
session._incrementUnacknowledgeMessageCount(message.fields.consumerTag);
decorateWithRoutingHeaders(message);
if (immediateNack(message)) { ackOrNack(session, message, new Error('Immediate nack')); return; }
decorateWithRedeliveries(message, (err) => {
  if (err) return handleRedeliveriesError(err, session, message);
  if (redeliveriesExceeded(message)) return handleRedeliveriesExceeded(session, message);
  getContent(message, config, (err, content) => {
    err ? handleContentError(session, message, err)
        : session.emit('message', message, content, getAckOrNack(session, message));
  });
});
```

The app receives `(message, content, ackOrNack)` via the `message` event.

### Content decoding / decryption [MUST]

`getContent`: if `headers.rascal.encryption` present, look up the named encryption profile in `config.encryption`, `decrypt(algorithm, key, iv, content)` (createDecipheriv), then negotiate using `config.contentType || originalContentType`. Otherwise negotiate using `config.contentType || message.properties.contentType`.

`negotiateContent`: `text/plain` → `content.toString()`; `application/json` → `safeParse` (safe-json-parse/callback); else raw buffer.

### Redeliveries [MUST]

- `decorateWithRedeliveries` — arms a `setTimeoutUnref` of `subscriptionConfig.redeliveries.timeout` (errors "Redeliveries timed out after Nms"), then `countRedeliveries`.
- `countRedeliveries` — if not `message.fields.redelivered` or no `messageId`, returns `0`; else `counter.incrementAndGet(\`${name}/${messageId}\`)`.
- `redeliveriesExceeded` — `redeliveries > subscriptionConfig.redeliveries.limit`.
- `handleRedeliveriesError` — emits `redeliveries_error` then `redeliveries_exceeded`; if neither handled, `nackAndError`.
- `handleRedeliveriesExceeded` — emits `redeliveries_exceeded` then `redeliveries_error`; else `nackAndError`.

### immediateNack (dead-letter replay handling) [MUST]

Returns true to immediately nack a message that carries a `rascal.recovery[originalQueue].immediateNack` header **unless** it has since been dead-lettered and replayed (detected by comparing the current `x-death` record's `count`/`time` against the stored `xDeath`). Handles RabbitMQ v3.12/v3.13 x-death quirks (issue #11331); uses `EMPTY_X_DEATH` fallback. On detecting a replay it unsets the `immediateNack`/`xDeath` recovery headers and returns false.

### decorateWithRoutingHeaders [MUST]

Sets `headers.rascal.originalQueue = subscriptionConfig.source` and `headers.rascal.originalVhost = vhost.name`. If `headers.rascal.restoreRoutingHeaders` is set, restores `fields.routingKey`/`fields.exchange` from the stored `originalRoutingKey`/`originalExchange`.

### ackOrNack contract [MUST]

`getAckOrNack` returns the promisified `ackOrNackP` only when `broker.promises && subscriptionConfig.promisifyAckOrNack`, else the callback `ackOrNack`.

**`ackOrNack(session, message, err, options, next)`** — heavily overloaded by arity:
- `(session, message)` → ack, error emitted on session via `emitOnError`.
- `(session, message, cb)` → ack with explicit callback.
- `(session, message, err)` → recover, default error emission.
- `(session, message, err, cb)` → recover with callback.
- `(session, message, err, options)` / `(…, options, next)` → recover with options.

Guards double-ack via `message.__rascal_acknowledged` (errors "ackOrNack should only be called once per message"). Then:
```js
if (err) return subscriberError.handle(session, message, err, options, next);   // recovery pipeline
if (options && options.all) return session._ackAll(message, next);
session._ack(message, next);
```

`ackOrNackP` is the Promise equivalent. `nackAndError` nacks then emits `error` on a `setTimeoutUnref` (so the nack IO completes before shutdown could roll the message back).

### Channel error / resubscription [MUST]

- `attachDisconnectionHandlers` — binds a `_.once` disconnection handler to channel `error` and connection `error`/`close`; returns a `removeDisconnectionHandlers` that also installs an error-swallowing handler on the cancelled channel.
- `handleChannelError` — emits `error`, `retrySubscription(attempt+1)`.
- `handleChannelClose` — emits `close`, `retrySubscription(attempt+1)`.
- `handleConsumerCancel` — broker-initiated cancel: emits `cancelled` → `cancel` → `error` (first handled wins), `session._close`, then `retrySubscription(1)`.
- `retrySubscription` — if `config.retry`, `subscribeNow`; on failure schedules a retried `handleChannelError` after `timer.next()` (backoff) via `session._schedule`.

### Events emitted (on SubscriberSession) [MUST]

| Event | Payload | When |
|-------|---------|------|
| `message` | `(message, content, ackOrNack)` | message delivered to app |
| `subscribed` | — | consume established |
| `error` | `err` | channel error, unhandled recovery, fatal content/redelivery errors |
| `close` | — | channel closed |
| `cancelled` | `err` | broker cancelled the consumer (preferred) |
| `cancel` | `err` | fallback if `cancelled` unhandled |
| `invalid_content` | `(err, message, ackOrNack)` | content decode error (legacy name) |
| `invalid_message` | `(err, message, ackOrNack)` | content decode error |
| `redeliveries_error` | `(err, message, ackOrNack)` | error counting redeliveries |
| `redeliveries_exceeded` | `(err, message, ackOrNack)` | redelivery limit exceeded |

For unhandled `invalid_*`/`redeliveries_*` events, the message is nacked and `error` emitted (`nackAndError`).

---

## SubscriberSession.js [MUST] <!-- id: file:lib/amqp/SubscriberSession.js:SubscriberSession.js -->

[SubscriberSession.js](https://github.com/cliftonc/rascal/blob/master/lib/amqp/SubscriberSession.js#L1-L229) · `inherits(SubscriberSession, EventEmitter)`

Tracks one or more consumer channels (keyed by `consumerTag`) and performs ack/nack against the right channel. Created with `(sequentialChannelOperations, config)`.

### Channel registry & open/close [MUST]

```js
this._open = (channel, consumerTag, next) => {
  if (cancelled) return next(Error('Subscriber has been cancelled'));
  channels[consumerTag] = { index: index++, channel, consumerTag, unacknowledgedMessages: 0 };
  channel.once('close', unref.bind(null, consumerTag));
  channel.once('error', unref.bind(null, consumerTag));
  next();
};
this.cancel = (next) => { clearTimeout(timeout); sequentialChannelOperations.push((done) => { cancelled = true; self._unsafeClose(done); }, next); };
this._close = (next) => sequentialChannelOperations.push((done) => self._unsafeClose(done), next);
this.isCancelled = () => cancelled;
this._schedule = (fn, delay) => { timeout = setTimeoutUnref(fn, delay); };
```

`_unsafeClose` — marks the current channel `doomed`, `channel.cancel(consumerTag)`, then waits for unacknowledged messages to drain (`waitForUnacknowledgedMessages`, polling every 100ms, optionally bounded by `config.closeTimeout` via `async.timeout`), then `channel.close`.

### setChannelPrefetch [MUST]

```js
this.setChannelPrefetch = (prefetch, next) => sequentialChannelOperations.push((done) => {
  config.channelPrefetch = prefetch;
  withCurrentChannel((channel) => channel.prefetch(prefetch, true, done), () => done());  // global=true
}, next);
```

### ack / nack [MUST]

```js
this._ack(message, next)       // channel.ack(message);          decrement count
this._ackAll(message, next)    // channel.ackAll();              reset count
this._nack(message, options, next)   // channel.nack(message, false, !!options.requeue); decrement
this._nackAll(message, options, next)// channel.nack(message, true,  !!options.requeue); reset
```

All select the channel via `withConsumerChannel(message.fields.consumerTag, …)`; if the channel is gone they `next(Error('The channel has been closed…'))`. All call `next` via `setImmediate`. Unacknowledged-message counts are maintained per channel (skipped entirely when `config.options.noAck`).

### Channel selection helpers [MUST]

- `withCurrentChannel(fn, altFn)` — picks the highest-`index`, non-`doomed` channel.
- `withConsumerChannel(consumerTag, fn, altFn)` — looks up by exact consumer tag.
- `unref(consumerTag)` — deletes a channel from the registry on its close/error.

---

## SubscriberSessionAsPromised.js [MUST] <!-- id: file:lib/amqp/SubscriberSessionAsPromised.js:SubscriberSessionAsPromised.js -->

[SubscriberSessionAsPromised.js](https://github.com/cliftonc/rascal/blob/master/lib/amqp/SubscriberSessionAsPromised.js#L1-L32) · `inherits(…, EventEmitter)`

Promise wrapper. `forwardEvents(session, this)` re-emits all session events. Exposes `name`, `config`, and promisified `cancel()` and `setChannelPrefetch(prefetch)`. Constructed by `BrokerAsPromised.subscribe`/`subscribeAll`.
