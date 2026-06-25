# Recovery

The subscriber error-recovery pipeline. When a consumer fails to process a message (or explicitly `ackOrNack`s with an error and recovery options), `SubscriberError.handle` walks an ordered list of recovery strategies, executing each in turn until one *handles* the message (acks/nacks it) or the list is exhausted and the implicit `fallback-nack` fires. `XDeath` provides the empty x-death sentinel used when counting dead-letter redeliveries during republish.

Link format: `https://github.com/cliftonc/rascal/blob/master/{path}#L{start}-L{end}`

## SubscriberError.js [MUST] <!-- id: file:lib/amqp/SubscriberError.js:SubscriberError.js -->

[SubscriberError.js](https://github.com/cliftonc/rascal/blob/master/lib/amqp/SubscriberError.js#L1-L216)

Constructed as `new SubscriptionRecovery(broker, vhost)`. Exposes a single public method, `handle(session, message, err, recoveryOptions, next)`, plus an internal keyed table of recovery strategies.

### handle(session, message, err, recoveryOptions, next) [MUST]

Orchestrates recovery by running the candidate strategies **in series** with `async.eachSeries`:

```js
async.eachSeries(
  [].concat(recoveryOptions || []).concat({ strategy: 'fallback-nack' }),
  (recoveryConfig, cb) => {
    const once = _.once(cb);
    setTimeoutUnref(() => {
      getStrategy(recoveryConfig).execute(session, message, err, _.omit(recoveryConfig, 'defer'), (err, handled) => {
        if (err)      { setImmediate(() => next(err)); return once(false); }
        if (handled)  { setImmediate(next);            return once(false); }
        once();
      });
    }, recoveryConfig.defer);
  },
  next,
);
```

**Order / fallthrough semantics:**

- The candidate list is `recoveryOptions` (an array, or a single object coerced via `[].concat`) **with an implicit `{ strategy: 'fallback-nack' }` appended** as a guaranteed terminal strategy. Recovery therefore can never silently do nothing — if all configured strategies decline, the message is nacked.
- Strategies run one at a time, in declared order.
- Each strategy's `execute` callback is `(err, handled)`:
  - **err** → recovery failed: `next(err)` is called (deferred via `setImmediate`) and the series is stopped (`once(false)` short-circuits `eachSeries`).
  - **handled truthy** → message consumed by this strategy: `next()` is called and the series stops.
  - **handled falsy / undefined** → strategy declined (e.g. attempt limit reached): `once()` advances to the **next** strategy in the list.
- `recoveryConfig.defer` is passed as the delay to `setTimeoutUnref` (an unref'd `setTimeout`, see `lib/amqp/utils/setTimeoutUnref`), so a strategy can be delayed without keeping the event loop alive. `defer` is stripped from the config (`_.omit(recoveryConfig, 'defer')`) before being handed to the strategy.
- `once = _.once(cb)` guards each strategy iteration against double-callback.
- `setImmediate` defers the `next` invocation out of the amqplib accept loop so thrown errors are not swallowed.

`getStrategy(recoveryConfig)` resolves `recoveryStrategies[recoveryConfig.strategy] || recoveryStrategies.unknown`, so an unrecognised strategy name falls through to the `unknown` strategy.

### Strategy: ack [MUST]

```js
const ackFn = strategyConfig.all ? session._nackAll.bind(session) : session._nack.bind(session);
ackFn(message, (err) => next(err, true));
```

Note (per source): the `ack` strategy is implemented with `session._nack` / `session._nackAll` (no `requeue` option), unconditionally returning `handled = true`. With `all: true` it nacks the entire prefetched batch; otherwise a single message. It always reports the message as handled, terminating the recovery chain.

### Strategy: nack [MUST]

```js
const nackFn = strategyConfig.all ? session._nackAll.bind(session) : session._nack.bind(session);
nackFn(message, { requeue: strategyConfig.requeue }, (err) => next(err, true));
```

Nacks the message (or the whole batch when `all: true`), honouring `strategyConfig.requeue` to control whether RabbitMQ requeues vs dead-letters. Always reports `handled = true`.

### Strategy: fallback-nack [MUST]

```js
session._nack(message, { requeue: strategyConfig.requeue }, (err) => next(err, true));
```

The terminal strategy appended automatically by `handle`. A plain single-message `_nack` honouring `requeue`, always `handled = true`. Guarantees the message is always acted upon even when no user strategy handled it.

### Strategy: republish [MUST]

Re-publishes the message back onto its **original queue** via a confirm channel, tracking attempt counts in the message's own `rascal` headers.

- Reads `originalQueue` from `properties.headers.rascal.originalQueue` and the prior `republished` count from `properties.headers.rascal.recovery[originalQueue].republished` (default `0`).
- **Attempt limit:** if `strategyConfig.attempts && strategyConfig.attempts <= republished`, returns `next(null, false)` — declines so the chain falls through to the next strategy.
- Builds `publishOptions` from a deep clone of `message.properties` and sets:
  - `headers.rascal.recovery[originalQueue].republished = republished + 1`
  - `headers.rascal.originalExchange` / `originalRoutingKey` from `message.fields`
  - `headers.rascal.error.message` (truncated to 1024 chars) and `headers.rascal.error.code`
  - `headers.rascal.restoreRoutingHeaders` (from config, default `true`)
- **immediateNack option:** when set, locates the matching `x-death` record (`queue === originalQueue && reason === 'rejected'`) or `EMPTY_X_DEATH`, and writes `headers.rascal.recovery[originalQueue] = { immediateNack: true, xDeath }`. This lets the subscription detect a dead-letter replay loop and immediately nack on next delivery.
- Acquires a confirm channel via `vhost.getConfirmChannel`. If the vhost is shutting down (`!publisherChannel`), acks-or-nacks with an error.
- Publishes with `publisherChannel.publish(undefined, originalQueue, message.content, publishOptions, cb)` (default exchange, routing key = queue name).
- On success: closes the channel, removes listeners, then **acks the original message** (`ackOrNack()`); on publish error or a `return`/channel `error` event: **nacks the original** (`ackOrNack(err)`), preserving the original message. `ackOrNack` is `_.once`-guarded.
- On the ack/nack path it ultimately calls `next(err, true)` (handled) on ack, or `next(err)` on nack.

[SubscriberError.js#L67-L141](https://github.com/cliftonc/rascal/blob/master/lib/amqp/SubscriberError.js#L67-L141)

### Strategy: forward [MUST]

Forwards the failed message to another **publication** (e.g. a dead-letter or retry exchange) via `broker.forward`, tracking a separate `forwarded` attempt count.

- Reads `originalQueue` and prior `forwarded` count from `properties.headers.rascal.recovery[originalQueue].forwarded` (default `0`).
- **Attempt limit:** if `strategyConfig.attempts && strategyConfig.attempts <= forwarded`, returns `next(null, false)` (declines, falls through).
- **xDeathFix option:** deletes `message.properties.headers['x-death']` before forwarding (workaround for rabbitmq-server issue #161).
- Builds `forwardOverrides` from a clone of `strategyConfig.options` and sets `restoreRoutingHeaders` (default `true`), the incremented `forwarded` count, and truncated error message/code under `options.headers.rascal`.
- Calls `broker.forward(strategyConfig.publication, message, forwardOverrides, cb)`:
  - on `forward` error → nacks the original (`nackMessage(err)`).
  - publication `success` → acks the original (handled).
  - publication `error` → nacks (`ackOrNack(err)`).
  - publication `return` → removes the `success` listener and nacks with a "forwarded but returned" error.
- `ackOrNack` is `_.once`-guarded; ack calls `next(err, true)`, nack calls `next(err)`.

[SubscriberError.js#L142-L200](https://github.com/cliftonc/rascal/blob/master/lib/amqp/SubscriberError.js#L142-L200)

### Strategy: unknown [MUST]

```js
session._nack(message, () => {
  next(new Error(format('Error recovering message: %s. No such strategy: %s.', message.properties.messageId, strategyConfig.strategy)));
});
```

The fallback returned by `getStrategy` when a configured `strategy` name is not in the table. Nacks the message and calls `next(err)` so the failure propagates. Since it errors rather than declining, it terminates the chain.

## XDeath.js [SHOULD] <!-- id: file:lib/amqp/XDeath.js:XDeath.js -->

[XDeath.js](https://github.com/cliftonc/rascal/blob/master/lib/amqp/XDeath.js#L1-L5)

```js
const EMPTY_X_DEATH = { count: 0, time: { value: 0 } };
module.exports = { EMPTY_X_DEATH };
```

Exports a single sentinel, `EMPTY_X_DEATH`, used as the default when no matching `x-death` header record is found while building the `immediateNack` recovery marker in the `republish` strategy (see above). RabbitMQ writes `x-death` header entries to dead-lettered messages recording the `count` of times a message was rejected from a given queue and the `time` of the most recent death; rascal reads these to count redeliveries/dead-letter replays. The sentinel provides a zero-count / zero-time baseline so downstream comparison logic (in `Subscription`'s `immediateNack` handling) has a well-formed shape even on the first pass.
