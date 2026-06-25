# Subscription Tests

The largest integration suites — they exercise consume + ack/nack + recovery semantics against a live broker. `beforeEach` builds a rich namespaced topology (exchanges incl. a dead-letter exchange `dlx`, queues incl. `dlq` and queues with periods in the name, plus subscriptions/publications) and a raw amqplib connection used to inject and inspect messages. Each consumed message is verified through both rascal's events and direct `amqputils` queue inspection.

## subscriptions.tests.js [MUST] <!-- id: file:test/subscriptions.tests.js:subscriptions.tests.js -->

[test/subscriptions.tests.js](https://github.com/cliftonc/rascal/blob/master/test/subscriptions.tests.js#L1-L2952)

Module under test: `require('..').Broker` (callback API). `beforeEach` ([L18](https://github.com/cliftonc/rascal/blob/master/test/subscriptions.tests.js#L18)) builds 4 exchanges (e1, e2, dlx, xx), 5 queues (q1, q2, q3, dlq, and a period-named queue), 5 bindings, 4 publications, 5 subscriptions, and connects via amqplib. ~74 `it` tests, grouped by behavior:

- **Basic consume / content types** ([L149+](https://github.com/cliftonc/rascal/blob/master/test/subscriptions.tests.js#L149)) — unknown subscription errors; consume text/plain, text/other (text/csv), arbitrary custom MIME, JSON (parsed), Buffer; messages not consumed before a listener is bound; force `contentType` when specified.
- **Repeated ackOrNack** ([L192+](https://github.com/cliftonc/rascal/blob/master/test/subscriptions.tests.js#L192)) — repeated `ackOrNack` reported via callback, or via `error` event when no callback.
- **Invalid content/message** ([L389+](https://github.com/cliftonc/rascal/blob/master/test/subscriptions.tests.js#L389)) — invalid messages silently discarded when no listener; delivered on `invalid_content` / `invalid_message`; consumed when a listener acks or nacks.
- **Routing key filter** ([L585](https://github.com/cliftonc/rascal/blob/master/test/subscriptions.tests.js#L585)).
- **Ack/nack strategies** ([L614+](https://github.com/cliftonc/rascal/blob/master/test/subscriptions.tests.js#L614)) — auto-ack (`noAck`), unacked messages remain queued, explicit ack, `all: true` ack; reject (nack) by default and reject-all; reject to dead-letter exchange.
- **Requeue** ([L894+](https://github.com/cliftonc/rascal/blob/master/test/subscriptions.tests.js#L894)) — requeue, requeue-all, deferred requeue.
- **Redeliveries** ([L1017+](https://github.com/cliftonc/rascal/blob/master/test/subscriptions.tests.js#L1017)) — count redeliveries (inMemory counter), notify on `redeliveries_exceeded` and `redeliveries_error`; poison message auto-acked or ack'd by a listener.
- **Republish strategy** ([L1240+](https://github.com/cliftonc/rascal/blob/master/test/subscriptions.tests.js#L1240)) — republish to queue (incl. queue names with periods); truncate error messages at 1024 chars; maintain fields/properties/headers; cap republishes (`attempts`); deferred republish.
- **Immediate nack** ([L1429+](https://github.com/cliftonc/rascal/blob/master/test/subscriptions.tests.js#L1429)) — `immediateNack`; ignore immediate nack for DLX-replayed messages (incl. repeated replays and period queue names).
- **Forward strategy** ([L1767+](https://github.com/cliftonc/rascal/blob/master/test/subscriptions.tests.js#L1767)) — forward to a publication; nack original on forward failure; period queue names; truncate errors; override routing key; maintain metadata; suppress original routing headers when requested; cap forwards; error forwarding to `/dev/null` (unroutable).
- **Recovery strategy chain** ([L2107+](https://github.com/cliftonc/rascal/blob/master/test/subscriptions.tests.js#L2107)) — error on unknown strategy; chain strategies; no rollback after ack on shutdown; nack when all strategies attempted (empty array).
- **Prefetch / concurrency** ([L2263+](https://github.com/cliftonc/rascal/blob/master/test/subscriptions.tests.js#L2263)) — consumer `prefetch`, `channelPrefetch`, and dynamic `setChannelPrefetch` during and before subscription.
- **Channel errors & lifecycle** ([L2450+](https://github.com/cliftonc/rascal/blob/master/test/subscriptions.tests.js#L2450)) — emit channel errors (PRECONDITION_FAILED on double ack); no consume after unsubscribe; tolerate repeated unsubscription; no emitter-leak warning with 11 subscriptions; attach subscription vhost (`originalVhost`) to properties; error acking/nacking after unsubscribe.
- **Encryption** ([L2665+](https://github.com/cliftonc/rascal/blob/master/test/subscriptions.tests.js#L2665)) — symmetric decrypt; `invalid_content` on missing profile; fail on encryption errors.
- **Broker cancellation** ([L2805+](https://github.com/cliftonc/rascal/blob/master/test/subscriptions.tests.js#L2805)) — emit `cancelled`/`cancel`; `error` when no cancel handler; resubscribe after broker cancellation (queue delete/recreate).

## subscriptionsAsPromised.tests.js [MUST] <!-- id: file:test/subscriptionsAsPromised.tests.js:subscriptionsAsPromised.tests.js -->

[test/subscriptionsAsPromised.tests.js](https://github.com/cliftonc/rascal/blob/master/test/subscriptionsAsPromised.tests.js#L1-L2135)

Module under test: `require('..').BrokerAsPromised` (promise API). `beforeEach` ([L17](https://github.com/cliftonc/rascal/blob/master/test/subscriptionsAsPromised.tests.js#L17)) builds exchanges e1/e2/xx, queues q1/q2/q3, 3 bindings, 3 publications, 4 subscriptions, and connects via amqplib. Suite timeout `{ timeout: 5000 }`. ~65 tests — a promise-API mirror of the callback suite covering the same themes: basic consume/content types, invalid content/message, content-type override, routing-key filter, ack/nack strategies, dead-letter reject, requeue (+all, deferred), redeliveries/limits/poison, republish (+truncate/metadata/cap/defer), immediate nack (+DLX replay), forward (+truncate/routingKey/metadata/cap/unroutable), recovery strategy chaining (+no-rollback-after-ack, nack-when-exhausted), prefetch/dynamic prefetch, channel errors, subscription lifecycle (no-consume-after-unsubscribe, repeated unsubscribe, no emitter leak), `originalVhost` property, ack/nack-after-unsubscribe rejections, and encryption (decrypt, missing profile, error).

Additional promise-specific cases ([L2067+](https://github.com/cliftonc/rascal/blob/master/test/subscriptionsAsPromised.tests.js#L2067)) — **Callback compatibility**: support `ackOrNack` using callbacks (`promisifyAckOrNack: false`); support handling recovery errors using callbacks.
