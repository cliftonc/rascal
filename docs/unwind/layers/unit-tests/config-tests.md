# Config Pipeline Tests

Three pure unit suites (no broker connection, no beforeEach/afterEach, no timeout override) that exercise the config pipeline modules directly: `configure`, `defaults`, and `validate`. They verify how raw user config is parsed, expanded/decorated, defaulted, and validated before any AMQP connection is made.

## config.tests.js [MUST] <!-- id: file:test/config.tests.js:config.tests.js -->

[test/config.tests.js](https://github.com/cliftonc/rascal/blob/master/test/config.tests.js#L1-L2105)

Module under test: `require('../lib/config/configure')`. ~77 `it` tests in nested describes. Verifies parsing + decoration of every config section.

- **Vhosts > Connection** ([L9](https://github.com/cliftonc/rascal/blob/master/test/config.tests.js#L9)) — build connection URL from object; parse string URL into connections array; merge options into a url; encode/decode url + query params; report invalid urls; decorate connection with vhost name (and empty pathname for `/`); generate loggable URL with masked/obscured passwords (connection + management); generate connections from an array; randomise connection order (100 iterations) while keeping order consistent across vhosts by host; honour order with `fixed` strategy; derive management url from connection url (default port 15672) or from explicit management config/object, reusing connection credentials; generate a namespace when `namespace: true`.
- **Vhosts > Exchanges** ([L682](https://github.com/cliftonc/rascal/blob/master/test/config.tests.js#L682)) — apply exchange config; add default `''` exchange (without overwriting a custom one); inflate empty structure; decorate with `name` and `fullyQualifiedName` (prefixed with namespace); support string-array and mixed array/object configuration.
- **Vhosts > Queues** ([L860](https://github.com/cliftonc/rascal/blob/master/test/config.tests.js#L860)) — apply queue config; inflate; decorate with name + FQN (namespace prefix); append uuid to `replyTo` queues; prefix `x-dead-letter-exchange` arg with namespace; mixed array/object config.
- **Vhosts > Bindings** ([L1013](https://github.com/cliftonc/rascal/blob/master/test/config.tests.js#L1013)) — apply binding config; parse `source -> destination` and `source[binding.key] -> destination` shorthand; qualify binding keys with namespace; inflate; decorate with name; expand an array of binding keys into multiple bindings (single-item array treated as scalar); mixed array/object config.
- **Publications** ([L1261](https://github.com/cliftonc/rascal/blob/master/test/config.tests.js#L1261)) — configure exchange/queue publications; inflate; decorate with name + destination; replace destination with FQN (namespace + uuid); default the publication vhost to the surrounding vhost; auto-create a default publication per exchange and per queue (and a default subscription per queue) without overriding explicit vhost/root-level definitions; merge implicit + explicit publications; hoist referenced encryption profiles; suffix referenced `replyTo` queue with uuid; report unknown `replyTo` queues.
- **Subscriptions** ([L1793](https://github.com/cliftonc/rascal/blob/master/test/config.tests.js#L1793)) — configure queue subscriptions; inflate; decorate with name + source; report duplicate subscription/publication names across vhosts; replace source with FQN; default vhost to surrounding block; merge implicit + explicit subscriptions; hoist encryption profiles.
- **Shovels** ([L2052](https://github.com/cliftonc/rascal/blob/master/test/config.tests.js#L2052)) — decorate shovel with name; parse `subscription -> publication` shorthand.
- **Redelivery Counters** ([L2084](https://github.com/cliftonc/rascal/blob/master/test/config.tests.js#L2084)) — decorate counter with name and type derived from key.

## defaults.tests.js [MUST] <!-- id: file:test/defaults.tests.js:defaults.tests.js -->

[test/defaults.tests.js](https://github.com/cliftonc/rascal/blob/master/test/defaults.tests.js#L1-L594)

Module under test: `require('../lib/config/configure')`. ~20 tests. Each describe asserts the same two-part contract: a default value is applied when config is empty, AND an explicit value overrides the default.

- **Vhosts > Connection** ([L6](https://github.com/cliftonc/rascal/blob/master/test/defaults.tests.js#L6)) — default vs overridden connection configuration.
- **Vhosts > Channel pooling** ([L78](https://github.com/cliftonc/rascal/blob/master/test/defaults.tests.js#L78)) — default vs overridden publication channel pool sizes.
- **Vhosts > Exchanges** ([L136](https://github.com/cliftonc/rascal/blob/master/test/defaults.tests.js#L136)), **Queues** ([L210](https://github.com/cliftonc/rascal/blob/master/test/defaults.tests.js#L210)), **Bindings** ([L283](https://github.com/cliftonc/rascal/blob/master/test/defaults.tests.js#L283)) — default vs overridden config for each.
- **Publications** ([L368](https://github.com/cliftonc/rascal/blob/master/test/defaults.tests.js#L368)) and **Subscriptions** ([L444](https://github.com/cliftonc/rascal/blob/master/test/defaults.tests.js#L444)) — default vs overridden.
- **Redeliveries** ([L533](https://github.com/cliftonc/rascal/blob/master/test/defaults.tests.js#L533)) — apply default config based on counter type; apply default config based on counter name.

## validation.tests.js [MUST] <!-- id: file:test/validation.tests.js:validation.tests.js -->

[test/validation.tests.js](https://github.com/cliftonc/rascal/blob/master/test/validation.tests.js#L1-L924)

Module under test: `require('../lib/config/validate')`. ~49 tests asserting validation errors are raised for malformed config.

- **Vhosts** ([L5](https://github.com/cliftonc/rascal/blob/master/test/validation.tests.js#L5)) — report zero vhosts.
- **Bindings** ([L19](https://github.com/cliftonc/rascal/blob/master/test/validation.tests.js#L19)) — mandate source, destination, destinationType; reject invalid destinationType; report unknown source/destination exchanges and destination queues (including empty-collection cases).
- **Publications** ([L223](https://github.com/cliftonc/rascal/blob/master/test/validation.tests.js#L223)) — mandate vhost; mandate exactly one of exchange/queue (neither, both, empty-string+queue cases); report unknown vhosts/exchanges/queues.
- **Subscriptions** ([L427](https://github.com/cliftonc/rascal/blob/master/test/validation.tests.js#L427)) — mandate vhost and queue; report unknown vhosts/queues/counters.
- **Shovels** ([L577](https://github.com/cliftonc/rascal/blob/master/test/validation.tests.js#L577)) — mandate subscription and publication; report unknown subscription/publication.
- **Vocabulary** ([L676](https://github.com/cliftonc/rascal/blob/master/test/validation.tests.js#L676)) — reject unsupported attributes on vhost, publication channel pool, connection (including unknown connection strategies), exchange, queue, binding, publication, subscription, shovel.
- **Encryption** ([L863](https://github.com/cliftonc/rascal/blob/master/test/validation.tests.js#L863)) — mandate key, algorithm, ivLength on encryption profiles.
