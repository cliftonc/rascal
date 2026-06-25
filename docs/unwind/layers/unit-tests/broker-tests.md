# Broker Tests

Integration tests for broker lifecycle (vhost assert/check/delete, nuke, bounce, purge, connect, subscribe-all, connection introspection). Run against a live broker + management API. The callback and promise suites mirror each other.

## broker.tests.js [MUST] <!-- id: file:test/broker.tests.js:broker.tests.js -->

[test/broker.tests.js](https://github.com/cliftonc/rascal/blob/master/test/broker.tests.js#L1-L540)

Module under test: `require('..').Broker` (callback API). `beforeEach` builds a per-test namespaced vhost config (e1/q1/b1, p1/p2, s1) and opens a raw amqplib connection wrapped in `amqputils`. `afterEach` disconnects + `broker.nuke`. Suite timeout `{ timeout: 6000 }`. `createBroker` supports an optional `components` argument (e.g. custom http agent).

Tests:
- should assert vhosts; should fail to assert vhost when unable to connect to management plugin (`ECONNREFUSED`); should fail when checking vhosts that dont exist (404 message); should fail to check vhost when unable to connect to management plugin; should succeed when checking vhosts that do exist; should delete vhosts (assert then nuke then check 404).
- should support custom http agent — injects an `agent` whose `createConnection` errors, asserting the error surfaces.
- should provide fully qualified name (`namespace:q1` via `getFullyQualifiedName`); should not modify configuration (JSON snapshot unchanged after create).
- should nuke; should tolerate unsubscribe timeouts when nuking (`ETIMEDOUT` via short `closeTimeout`).
- should cancel subscriptions; should not return from unsubscribeAll until underlying channels have been closed (defers ~200ms, `ETIMEDOUT`).
- should connect (`connection._rascal_id` set, then closes).
- should tolerate unsubscribe timeouts when shuting down (`shutdown`); should bounce vhosts; should tolerate unsubscribe timeouts when bouncing.
- should purge vhosts (publish then purge then assert message absent).
- should emit busy/ready events (skipped on CI; publishes fast to trigger `busy`, pauses, then `ready`; timeout 60000).
- should subscribe to all subscriptions (`subscribeAll` returns 2 `SubscriberSession`s named `s1`, `/q1`); should subscribe to all filtered subscriptions (filter out `autoCreated` -> 1).
- should get vhost connections (`getConnections()` length 1, vhost `/`, masked `connectionUrl` `amqp://guest:***@localhost:5672?heartbeat=50&connection_timeout=10000&channelMax=100`).

## brokerAsPromised.tests.js [MUST] <!-- id: file:test/brokerAsPromised.tests.js:brokerAsPromised.tests.js -->

[test/brokerAsPromised.tests.js](https://github.com/cliftonc/rascal/blob/master/test/brokerAsPromised.tests.js#L1-L247)

Module under test: `require('..').BrokerAsPromised` (promise API). Same per-test vhost config; `afterEach` returns `broker.nuke()`. Suite timeout `{ timeout: 6000 }`. `createBroker` returns a promise and captures the broker even from a rejected `create` (`err.broker`).

Promise-API mirror of the callback suite (subset):
- should assert vhosts; should fail when checking vhosts that dont exist (`.catch` asserts 404 message); should not fail when checking vhosts that do exist; should delete vhosts.
- should provide fully qualified name; should not modify configuration; should nuke.
- should cancel subscriptions; should connect (`_rascal_id`).
- should subscribe to all subscriptions (2 `SubscriberSessionAsPromised`s, names `s1`, `/q1`); should subscribe to all filtered subscriptions (1).
- should get vhost connections (same masked `connectionUrl` assertion).
