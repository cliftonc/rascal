# Publication Tests

Integration tests for publish behavior. `beforeEach` builds a namespaced vhost (exchanges e1/e2/xx, queues q1/q2/q3, bindings b1/b2) and a raw amqplib connection. Assertions use `amqputils.assertMessage` / `getMessage` against the queue to confirm the message actually landed with the right content, properties, and headers. Suite timeout `{ timeout: 2000 }`.

## publications.tests.js [MUST] <!-- id: file:test/publications.tests.js:publications.tests.js -->

[test/publications.tests.js](https://github.com/cliftonc/rascal/blob/master/test/publications.tests.js#L1-L916)

Module under test: `require('..').Broker` (callback API).

- should report unknown publications (`Unknown publication: ...`); should report deprecated publications (still publishes).
- should publish text messages to normal exchanges (confirm:false); ... using confirm channels to exchanges (confirm:true); ... to queues; ... to queues via the default exchange (`broker.qualify`).
- should decorate the message with a uuid (messageId matches uuid regex and equals message's `properties.messageId`); should honour messageId when specified (`wibble`); should decorate error events with messageId.
- should publish using confirm channels to queues; should publish json messages (content equals `JSON.stringify`); should publish messages with custom contentType; should publish buffer messages.
- should allow publish overrides (`expiration: 1` -> message absent after 100ms); should report unrouted messages (`return` event on unbound exchange `xx`).
- should forward messages to publications (`broker.forward`; asserts routingKey, messageId, contentType, content, and `headers.rascal` originalQueue/originalRoutingKey/originalExchange + `restoreRoutingHeaders: false`); should instruct subscriber to restore routing headers when requested (`restoreRoutingHeaders: true`); should forward maintaining the original routing key when not overriden.
- should publish lots of messages using normal channels (1000, timeout 60000); ... using confirm channels (1000, timeout 20000).
- should symetrically encrypt messages (aes-256-cbc; asserts `headers.rascal.encryption` name/iv length/originalContentType and `contentType: application/octet-stream`); should report encryption errors (`Invalid key length`).
- should capture publication stats for normal channels / for confirm channels (`publication.stats.duration` is a non-negative number).
- should publish large messages to exchanges / to queues when using confirm channels (20MB buffers, single-slot confirm pool).
- should set the replyTo property (FQN `namespace:rq:replyTo` pattern).

## publicationsAsPromised.tests.js [MUST] <!-- id: file:test/publicationsAsPromised.tests.js:publicationsAsPromised.tests.js -->

[test/publicationsAsPromised.tests.js](https://github.com/cliftonc/rascal/blob/master/test/publicationsAsPromised.tests.js#L1-L582)

Module under test: `require('..').BrokerAsPromised` (promise API). Same vhost fixture (queues q1/q2). Promise-API mirror of the callback suite; for event-driven cases it still uses `(test, done)`.

Covers (via `.then()/.catch()`): unknown publications; deprecated publications; text to exchanges (confirm false/true); text to queues; decorate with uuid; confirm channels to queues; json; custom contentType; buffer; publish overrides (`expiration`); unrouted (`return` event); forward (full header assertions) and forward maintaining original routing key; publish 1000 messages on normal (timeout 60000) and confirm (timeout 20000) channels (sequential promise reduction); symmetric encryption; encryption errors (`Invalid key length`); publication stats for normal/confirm channels; set the replyTo property. Suite timeout `{ timeout: 2000 }`.
