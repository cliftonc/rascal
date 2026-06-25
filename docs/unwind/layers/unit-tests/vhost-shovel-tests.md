# Vhost & Shovel Tests

## vhost.tests.js [MUST] <!-- id: file:test/vhost.tests.js:vhost.tests.js -->

[test/vhost.tests.js](https://github.com/cliftonc/rascal/blob/master/test/vhost.tests.js#L1-L287)

Module under test: `require('..').Broker` (callback API). Integration test for vhost topology creation and connection lifecycle. `beforeEach` opens a raw amqplib connection wrapped by `amqputils`; `afterEach` disconnects + `broker.nuke`. Suite timeout `{ timeout: 2000 }`. Each test calls `createBroker` with a minimal per-test config (unique `namespace = uuid()`) merged over `testConfig`.

- should timeout connections — connects to unreachable host `10.255.255.1` with `socketOptions.timeout: 100`, asserts `connect ETIMEDOUT`.
- should create exchanges — asserts exchange present via `amqputils.assertExchangePresent`.
- should create objects concurrently — skipped on CI; creates 100 exchanges/queues/bindings serially (`concurrency: 1`) vs concurrently (`concurrency: 10`) over 5 reps and asserts the concurrent average is < half the serial average; per-test timeout 60000.
- should create queues — `assertQueuePresent`.
- should fail when checking a missing exchange / missing queue — `assert: false, check: true` yields a `NOT-FOUND` error.
- should create bindings — exchange-to-exchange (`destinationType: 'exchange'`) and exchange-to-queue bindings; publishes via `amqputils` and asserts the message reaches `q1`.
- should reconnect on error — uses `amqputils.waitForConnections` + `closeConnections` (management API) to force a `CONNECTION-FORCED` disconnect, asserts the broker emits `error` then re-emits `connect`; per-test timeout 10000.

## shovel.tests.js [MUST] <!-- id: file:test/shovel.tests.js:shovel.tests.js -->

[test/shovel.tests.js](https://github.com/cliftonc/rascal/blob/master/test/shovel.tests.js#L1-L104)

Module under test: `require('..').Broker` (callback API). Integration test for shovels (subscription -> publication transfer). `beforeEach` defines a namespaced config: exchanges e1/e2, queues q1/q2, bindings b1(`e1[foo]->q1`)/b2(`e2[bar]->q2`), publications p1(e1/foo) and p2(e2/bar), subscriptions s1(q1) and s2(q2, noAck), and a shovel `x1` wiring subscription `s1` to publication `p2`. `afterEach` nukes the broker. Note: this suite does NOT open a separate amqplib connection — it observes via rascal's own subscription.

- should transfer message from subscriber to publication — publishes to `p1` (lands in q1, consumed by shovel `s1`), the shovel republishes via `p2` to q2, and a subscription on `s2` receives the message, completing the test.
