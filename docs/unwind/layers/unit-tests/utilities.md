# Test Utilities

## amqputils [MUST] <!-- id: file:test/utils/amqputils.js:amqputils.js -->

[test/utils/amqputils.js](https://github.com/cliftonc/rascal/blob/master/test/utils/amqputils.js#L1-L162)

Factory `init(connection)` wrapping a raw `amqplib` connection, used by every integration suite to introspect the broker independently of rascal. All name-taking helpers prepend `${namespace}:` to mirror rascal's namespacing, and most are `_.curry`-ed so suites can partially apply them.

Exposed API (returned object):

| Helper | Purpose / assertion |
|--------|---------------------|
| `disconnect(next)` | closes the underlying connection |
| `checkExchange(present, name, namespace, next)` | `checkExchange`; asserts presence matches `present` |
| `createQueue(name, namespace, next)` | `assertQueue` to create a queue |
| `checkQueue(present, name, namespace, next)` | `checkQueue`; asserts presence matches `present` |
| `deleteQueue(name, namespace, next)` | `deleteQueue` |
| `publishMessage(exchange, namespace, message, options, next)` | publish to a (namespaced) exchange |
| `publishMessageToQueue(queue, namespace, message, options, next)` | publish to a queue via default exchange (routingKey = namespaced queue) |
| `getMessage(queue, namespace, next)` | `channel.get` with `noAck:true` |
| `assertMessage(queue, namespace, expected, next)` | asserts a message is present and its content equals `expected` |
| `assertMessageAbsent(queue, namespace, next)` | asserts no message present |
| `assertExchangePresent` / `assertExchangeAbsent` | `checkExchange` bound to `true` / `false` |
| `assertQueuePresent` / `assertQueueAbsent` | `checkQueue` bound to `true` / `false` |
| `waitForConnections(next)` | polls management API (up to 100 x 100ms) until at least one connection exists |
| `fetchConnections(next)` | GET `http://guest:guest@localhost:15672/api/connections` |
| `closeConnections(connections, reason, next)` | DELETEs each connection via management API (used to simulate broker-forced disconnects in recovery tests) |
| `closeConnection(name, reason, next)` | DELETE a single connection with `x-reason` header |

The management-API helpers (`waitForConnections`, `closeConnections`) are what let the vhost / subscription recovery tests forcibly drop the broker connection and assert reconnection.

## lib/config/tests.js (shared base config)

[lib/config/tests.js](https://github.com/cliftonc/rascal/blob/master/lib/config/tests.js)

Not a test file but the shared fixture merged into every integration suite. Documented in [harness.md](harness.md#shared-test-config). Defines ephemeral, non-durable, purge-on-create, namespaced topology defaults plus an in-memory redelivery counter size of 1000.
