# Architecture Discovery: rascal

> **For Claude:** REQUIRED SUB-SKILL: Use unwind:uw-analyze to analyze each layer.

## Discovery Metadata

- **Generated:** 2026-06-25T00:00:00Z
- **Project Root:** /Users/clifton/work/rascal
- **Framework:** amqplib-based RabbitMQ client library
- **Language:** JavaScript (Node.js)
- **Total Files:** 105 (79 with JS symbols)

## Repository Information

```yaml
repository:
  type: github
  url: https://github.com/cliftonc/rascal
  branch: master
  link_format: https://github.com/cliftonc/rascal/blob/master/{path}#L{start}-L{end}
```

**Note for downstream agents:** Use `link_format` to create source code links.

## Layer Configuration

```yaml
layers:
  database:
    status: not_detected
    confidence: high
    entry_points: []
    dependencies: []
    notes: No persistent database — rascal is a messaging client library.

  domain_model:
    status: detected
    confidence: high
    entry_points:
      - lib/config/baseline.js
      - lib/config/fqn.js
    dependencies: []
    notes: |
      Configuration schema and fully-qualified naming (FQN) rules form the domain model.
      Config objects (vhost, exchange, queue, binding, publication, subscription) are
      the primary domain entities; fqn.js provides namespace/uniqueness semantics.

  infrastructure_config:
    status: detected
    confidence: high
    entry_points:
      - lib/config/configure.js
      - lib/config/validate.js
      - lib/config/defaults.js
      - lib/config/schema.json
    dependencies: [domain_model]
    notes: |
      Configuration pipeline: baseline defaults → user config merge → validation → FQN enrichment.
      validate.js ensures schema compliance; configure.js applies defaults, parses connection URLs,
      and computes fully-qualified names for topology objects.

  messaging:
    status: detected
    confidence: high
    entry_points:
      - lib/amqp/Broker.js
      - lib/amqp/BrokerAsPromised.js
      - lib/amqp/Vhost.js
      - lib/amqp/Publication.js
      - lib/amqp/Subscription.js
      - lib/amqp/PublicationSession.js
      - lib/amqp/SubscriberSession.js
      - lib/amqp/SubscriberSessionAsPromised.js
      - lib/amqp/SubscriberError.js
      - lib/amqp/XDeath.js
      - lib/amqp/tasks/
    dependencies: [infrastructure_config, service_layer]
    notes: |
      Core RabbitMQ abstraction layer. Broker is the root orchestrator; Vhost manages
      connection pooling (via generic-pool) and channel lifecycle. Publication/Subscription
      wrappers provide async callback and Promise APIs over amqplib channels.
      26 task files compose the topology assertion/initialization pipeline using async.compose:
      assertVhost → createConnection → createChannels → assertExchanges → assertQueues →
      applyBindings → checkQueues → etc. Recovery strategies (SubscriberError) handle
      message redelivery with configurable backoff and retry logic.

  service_layer:
    status: detected
    confidence: high
    entry_points:
      - lib/backoff/
      - lib/counters/
    dependencies: []
    notes: |
      Backoff strategies (exponential, linear) for connection retry and subscription recovery.
      Counter implementations (stub, inMemory, inMemoryCluster) track message redelivery counts.
      These are composable, injected into Broker via components parameter.

  api:
    status: detected
    confidence: high
    entry_points:
      - index.js
      - lib/management/Client.js
    dependencies: [messaging, service_layer]
    notes: |
      Public library surface (index.js): exports Broker, BrokerAsPromised, createBroker,
      createBrokerAsPromised, config helpers (withDefaultConfig, withTestConfig), counters.
      Management Client: thin HTTP wrapper around RabbitMQ management API for vhost
      assert/check/delete operations (used by tasks).

  frontend:
    status: not_detected
    confidence: high
    entry_points: []
    dependencies: []
    notes: No UI — rascal is a backend messaging library.

cross_cutting:
  logging:
    touches: [messaging, service_layer, infrastructure_config]
    entry_points: [debug module throughout]
    notes: debug() calls throughout; no centralized logger

  event_emission:
    touches: [messaging]
    entry_points:
      - lib/amqp/Broker.js
      - lib/amqp/Vhost.js
      - lib/amqp/Publication.js
      - lib/amqp/Subscription.js
      - lib/amqp/PublicationSession.js
      - lib/amqp/SubscriberSession.js
    notes: All core classes inherit from EventEmitter; forward-emitter propagates events from Vhost to Broker

  async_patterns:
    touches: [messaging, service_layer, infrastructure_config]
    entry_points: [async module (async.compose, async.queue, async.eachSeries, async.retry)]
    notes: async.compose chains task pipelines; async.queue serializes channel operations; callbacks throughout

  error_handling:
    touches: [messaging]
    entry_points: [lib/amqp/SubscriberError.js]
    notes: Multi-level recovery strategies with configurable defer, fallback-nack, and custom handlers

  utilities:
    touches: [messaging, service_layer]
    entry_points: [lib/utils/setTimeoutUnref.js]
    notes: setTimeoutUnref uses unref'd timers to avoid holding the process alive
```

## Domain Model

**Status:** Detected | **Confidence:** High

**Entry Points:**
- [lib/config/baseline.js](https://github.com/cliftonc/rascal/blob/master/lib/config/baseline.js)
- [lib/config/fqn.js](https://github.com/cliftonc/rascal/blob/master/lib/config/fqn.js)
- [lib/config/defaults.js](https://github.com/cliftonc/rascal/blob/master/lib/config/defaults.js)

**Initial Observations:**

Rascal's domain model is configuration-centric. Users define RabbitMQ topology (vhosts, exchanges, queues, bindings) and messaging contracts (publications, subscriptions) as deeply-nested configuration objects. The domain entities include:

1. **Vhost** — a logical partitioning of the RabbitMQ broker, with its own exchange/queue/binding topology
2. **Exchange** — named routing destinations (topic, fanout, direct, headers)
3. **Queue** — named message buffers with optional dead-letter exchanges and redelivery policies
4. **Binding** — edges connecting exchanges to queues via routing keys
5. **Publication** — outbound messaging contract specifying target exchange/queue, options (persistent, mandatory, encryption), retry/timeout behavior
6. **Subscription** — inbound messaging contract specifying source queue, consumer prefetch, recovery/retry strategies
7. **Shovel** — internal message bridge (publication → subscription pipeline)
8. **Encryption Profile** — named cipher configurations for end-to-end message encryption
9. **Redelivery Counter** — tracks message redelivery attempts (stub, inMemory, inMemoryCluster implementations)

The **fully-qualified name (FQN)** pattern in [fqn.js](https://github.com/cliftonc/rascal/blob/master/lib/config/fqn.js) applies namespace prefixes and uniqueness suffixes to entity names, enabling multi-tenancy and test isolation. Baseline defaults ([baseline.js](https://github.com/cliftonc/rascal/blob/master/lib/config/baseline.js)) establish sensible topology and connection retry policies.

---

## Infrastructure & Configuration

**Status:** Detected | **Confidence:** High

**Entry Points:**
- [lib/config/configure.js](https://github.com/cliftonc/rascal/blob/master/lib/config/configure.js) — configuration normalization pipeline
- [lib/config/validate.js](https://github.com/cliftonc/rascal/blob/master/lib/config/validate.js) — schema validation
- [lib/config/tests.js](https://github.com/cliftonc/rascal/blob/master/lib/config/tests.js) — test defaults

**Initial Observations:**

Configuration is a three-stage pipeline:

1. **Baseline merge** — [baseline.js](https://github.com/cliftonc/rascal/blob/master/lib/config/baseline.js) sets connection, vhost, exchange, queue, binding, publication, and subscription defaults (e.g., 'topic' exchange type, exponential retry backoff min: 1000ms, max: 60000ms).

2. **Validation** ([validate.js](https://github.com/cliftonc/rascal/blob/master/lib/config/validate.js)) — enforces required fields (vhosts must exist, publications/subscriptions must reference valid vhosts), validates connection strategy ('random', 'fixed', or undefined), ensures no unsupported attributes.

3. **Normalization** ([configure.js](https://github.com/cliftonc/rascal/blob/master/lib/config/configure.js)) — parses connection URLs into protocol/hostname/user/password/port/vhost/options; applies namespace prefixes; computes FQNs; sets up management connection details (for vhost HTTP API assertions).

The Broker uses [async.compose](https://github.com/cliftonc/rascal/blob/master/lib/amqp/Broker.js) to chain validate → configure, ensuring immutability via lodash cloneDeep. Configuration is stored on the Broker instance for introspection.

---

## Messaging Layer

**Status:** Detected | **Confidence:** High

**Entry Points:**
- [lib/amqp/Broker.js](https://github.com/cliftonc/rascal/blob/master/lib/amqp/Broker.js) — root orchestrator
- [lib/amqp/BrokerAsPromised.js](https://github.com/cliftonc/rascal/blob/master/lib/amqp/BrokerAsPromised.js) — Promise wrapper
- [lib/amqp/Vhost.js](https://github.com/cliftonc/rascal/blob/master/lib/amqp/Vhost.js) — connection and channel pool management
- [lib/amqp/Publication.js](https://github.com/cliftonc/rascal/blob/master/lib/amqp/Publication.js) — publish abstraction
- [lib/amqp/Subscription.js](https://github.com/cliftonc/rascal/blob/master/lib/amqp/Subscription.js) — subscribe abstraction
- [lib/amqp/tasks/](https://github.com/cliftonc/rascal/blob/master/lib/amqp/tasks/) — 26 topology/lifecycle task modules

**Initial Observations:**

**Broker architecture:**

[Broker.js](https://github.com/cliftonc/rascal/blob/master/lib/amqp/Broker.js) is an EventEmitter that coordinates initialization, publication, subscription, and lifecycle. The constructor uses `async.compose` to chain five initialization tasks:

```
init = async.compose(initShovels, initSubscriptions, initPublications, initCounters, initVhosts)
```

Each task is curried and expects (config, context, callback). This pipeline ensures vhosts connect first, then publications and subscriptions are wired up, finally shovels (bridges) are established. The Broker maintains internal registries (vhosts, publications, subscriptions, sessions) and emits lifecycle events ('error', 'vhost_initialised').

[BrokerAsPromised.js](https://github.com/cliftonc/rascal/blob/master/lib/amqp/BrokerAsPromised.js) wraps the callback-based Broker with Promise-returning methods (connect, nuke, purge, shutdown, bounce, publish, subscribe, subscribeAll, unsubscribeAll) and forwards EventEmitter events.

**Vhost & connection pooling:**

[Vhost.js](https://github.com/cliftonc/rascal/blob/master/lib/amqp/Vhost.js) manages a single vhost's lifecycle. It maintains two generic-pool channel pools (regularPool, confirmPool for publisher-confirms) and uses `async.compose` to chain topology assertion tasks:

```
init = async.compose(closeChannels, applyBindings, purgeQueues, checkQueues, assertQueues, checkExchanges, assertExchanges, createChannels, createConnection, checkVhost, assertVhost)
```

The Vhost emits 'connect', 'vhost_initialised', 'paused' (when channel allocation pauses), and 'disconnect' events. Disconnection triggers automatic reconnection with exponential backoff via a backoff timer. A channel queue (async.queue with concurrency 1) serializes channel creation to avoid contention.

**Publication & subscription abstractions:**

[Publication.js](https://github.com/cliftonc/rascal/blob/master/lib/amqp/Publication.js) wraps two amqplib channel operations: publishToExchange (channel.publish) or sendToQueue (channel.sendToQueue), with or without publisher confirms. The publish method accepts payload/overrides and applies encryption if configured. [PublicationSession.js](https://github.com/cliftonc/rascal/blob/master/lib/amqp/PublicationSession.js) tracks individual publish operations (started, duration, stats) and emits 'paused' events when the vhost pauses.

[Subscription.js](https://github.com/cliftonc/rascal/blob/master/lib/amqp/Subscription.js) wraps amqplib channel.consume. The subscribe method returns a [SubscriberSession](https://github.com/cliftonc/rascal/blob/master/lib/amqp/SubscriberSession.js) EventEmitter that manages message delivery. Lazy subscription (subscribeLater) defers channel.consume until the first 'message' listener is attached. Messages flow through [SubscriberError.js](https://github.com/cliftonc/rascal/blob/master/lib/amqp/SubscriberError.js) error handling, which chains recovery strategies (ack, nack, fallback-nack, forward, dlq) with configurable defer times. [SubscriberSessionAsPromised.js](https://github.com/cliftonc/rascal/blob/master/lib/amqp/SubscriberSessionAsPromised.js) wraps the callback session with Promise methods.

**Tasks pipeline:**

26 task files in [lib/amqp/tasks/](https://github.com/cliftonc/rascal/blob/master/lib/amqp/tasks/) compose the topology assertion and lifecycle workflows:

- **Connection/channel:** createConnection, closeConnection, createChannels, closeChannels
- **Topology assertion:** assertVhost, assertExchanges, assertQueues, applyBindings
- **Topology checks:** checkVhost, checkExchanges, checkQueues
- **Topology cleanup:** deleteVhost, deleteExchanges, deleteQueues, purgeQueues, purgeVhost
- **Initialization:** initVhosts, initPublications, initSubscriptions, initCounters, initShovels
- **Lifecycle:** forewarnVhost, shutdownVhost, bounceVhost, nukeVhost

Each task is a curried function: (config, context, callback) → callback(err, config, context). The context accumulates mutable state (vhost objects, channel pools, connection details). Tasks use `async.retry` to handle connection failures by cycling through configured connection URLs.

---

## Service Layer

**Status:** Detected | **Confidence:** High

**Entry Points:**
- [lib/backoff/index.js](https://github.com/cliftonc/rascal/blob/master/lib/backoff/index.js) — backoff strategy factory
- [lib/backoff/exponential.js](https://github.com/cliftonc/rascal/blob/master/lib/backoff/exponential.js) — exponential backoff
- [lib/backoff/linear.js](https://github.com/cliftonc/rascal/blob/master/lib/backoff/linear.js) — linear backoff
- [lib/counters/index.js](https://github.com/cliftonc/rascal/blob/master/lib/counters/index.js) — counter implementations
- [lib/counters/stub.js](https://github.com/cliftonc/rascal/blob/master/lib/counters/stub.js) — no-op counter
- [lib/counters/inMemory.js](https://github.com/cliftonc/rascal/blob/master/lib/counters/inMemory.js) — single-process counter
- [lib/counters/inMemoryCluster.js](https://github.com/cliftonc/rascal/blob/master/lib/counters/inMemoryCluster.js) — multi-process counter

**Initial Observations:**

**Backoff strategies:**

The backoff module factory ([index.js](https://github.com/cliftonc/rascal/blob/master/lib/backoff/index.js)) returns a backoff timer object with methods like reset(), pause(), resume(). Options include:

- `delay` — use linear strategy with min: delay
- `strategy: 'exponential'` (default) — apply exponential backoff (min, max, factor)
- `strategy: 'linear'` — apply linear backoff (min, max, factor)

Used by [Vhost.js](https://github.com/cliftonc/rascal/blob/master/lib/amqp/Vhost.js) for connection retry (timer = backoff(connectionConfig.retry)) and [Subscription.js](https://github.com/cliftonc/rascal/blob/master/lib/amqp/Subscription.js) for subscription recovery (timer = backoff(subscriptionConfig.retry)).

**Counter implementations:**

Counters track message redelivery counts for dead-letter queue (DLQ) routing decisions:

- **stub** — no-op; always returns 0 (suitable for testing)
- **inMemory** — in-process Map keyed by message ID (not suitable for multi-process)
- **inMemoryCluster** — uses node cluster module to forward count queries to the master process
- Injected into Broker via components.counters, exposed in index.js exports

---

## API (Public Surface & Management Client)

**Status:** Detected | **Confidence:** High

**Entry Points:**
- [index.js](https://github.com/cliftonc/rascal/blob/master/index.js) — package entry point
- [lib/management/Client.js](https://github.com/cliftonc/rascal/blob/master/lib/management/Client.js) — RabbitMQ management API client

**Initial Observations:**

**Public API surface:**

[index.js](https://github.com/cliftonc/rascal/blob/master/index.js) exports:

- `Broker` — callback-based root class
- `BrokerAsPromised` — Promise-based wrapper
- `createBroker(config, components, callback)` — factory function
- `createBrokerAsPromised(config, components)` — Promise factory function
- `defaultConfig` — baseline configuration object
- `testConfig` — test-optimized configuration object
- `withDefaultConfig(config)` — merge user config with defaults
- `withTestConfig(config)` — merge user config with test defaults
- `counters` — { stub, inMemory, inMemoryCluster }

Typical usage:

```javascript
const rascal = require('rascal');
const config = rascal.withDefaultConfig(userConfig);
rascal.createBrokerAsPromised(config).then(broker => {
  broker.subscribe('mySubscription').then(session => {
    session.on('message', message => { /* ... */ });
  });
  broker.publish('myPublication', { data: 'payload' });
});
```

**Management Client:**

[lib/management/Client.js](https://github.com/cliftonc/rascal/blob/master/lib/management/Client.js) is a thin HTTP client for the RabbitMQ management REST API. Methods:

- `assertVhost(name, config, callback)` — PUT /api/vhosts/{name}
- `checkVhost(name, config, callback)` — GET /api/vhosts/{name}
- `deleteVhost(name, config, callback)` — DELETE /api/vhosts/{name}

Used by tasks (e.g., [assertVhost.js](https://github.com/cliftonc/rascal/blob/master/lib/amqp/tasks/assertVhost.js)) with async.retry cycling through connection URLs. Detects HTTP status >= 300 as failure.

---

## Cross-Cutting Concerns

**Logging:**
- **Touches:** messaging, infrastructure_config, service_layer
- **Entry Points:** debug() calls throughout (e.g., `debug('rascal:Broker')`)
- **Technology:** debug module with namespaced loggers; no centralized configuration

**Event Emission:**
- **Touches:** messaging
- **Entry Points:** Broker, Vhost, Publication, Subscription, PublicationSession, SubscriberSession (all inherit EventEmitter)
- **Technology:** Node.js events.EventEmitter; forward-emitter for event propagation (Vhost → Broker)
- **Key Events:** 'connect', 'vhost_initialised', 'disconnect', 'error', 'paused', 'subscribed', 'message'

**Async Control Flow:**
- **Touches:** messaging, infrastructure_config, service_layer
- **Entry Points:** async module (async.compose, async.queue, async.eachSeries, async.retry)
- **Pattern:** Curried callbacks; chained task composition; task context accumulation
- **Note:** No Promises at the core layer (callback-based); BrokerAsPromised wraps for end users

**Error Handling & Recovery:**
- **Touches:** messaging (Subscription)
- **Entry Points:** SubscriberError.js, recovery strategies
- **Pattern:** Configurable multi-level recovery: ack, nack (with requeue flag), forward (to another exchange), dlq (dead-letter queue), fallback-nack. Each strategy is deferred by a configurable delay.

**Utilities:**
- **Touches:** messaging, service_layer
- **Entry Points:** lib/utils/setTimeoutUnref.js
- **Technology:** Unref'd timers (setTimeout(...).unref()) to prevent process hang when listeners exist

---

## Infrastructure

**Build & CI:**
- **Package Manager:** npm (package.json defines scripts: test, lint, lint:fix, coverage, docker)
- **Test Runner:** zUnit (configured in package.json; pattern `^[\w-]+.tests.js$`)
- **Linter:** eslint with airbnb-base config
- **Git Hooks:** husky for pre-commit linting (lint-staged)
- **Docker:** RabbitMQ 3.12.9-management-alpine for local development

**Configuration Files:**
- `.npmrc` — npm registry/auth configuration
- `.husky/pre-commit` — git hook for lint-staged
- `.github/workflows/` — GitHub Actions CI pipelines (node-js-ci, node-js-publish)

**Schema:**
- [lib/config/schema.json](https://github.com/cliftonc/rascal/blob/master/lib/config/schema.json) — JSON Schema for configuration validation (referenced by validate.js)

---

## Tests & Examples

**Tests:**
- **Location:** test/ (15 test files following `*.<test-suite>.tests.js` pattern)
- **Runner:** zUnit (configured in package.json zUnit.pattern)
- **Coverage Tool:** nyc (npm run coverage)
- **Subjects:** backoff strategies, message handling, config validation, error recovery, vhost lifecycle, publications, subscriptions, shovels

**Examples:**
- **Location:** examples/ (12 files across 7 subdirectories)
- **Subdirectories:**
  - simple/ — basic pub/sub
  - promises/ — Promise API usage
  - default-exchange/ — routing to default exchange
  - streams/ — stream integration
  - busy-publisher/ — stress test
  - advanced/ — recovery and DLQ patterns
  - mocha/ — Mocha test integration
- **Purpose:** Documentation and demonstration of public API

---

## Discovery Notes

1. **Callback-First Architecture:** Rascal's core is callback-based (async.compose, async.queue); BrokerAsPromised is a thin Promise wrapper for end users.

2. **Event-Driven Lifecycle:** Heavy use of EventEmitter for vhost/publication/subscription lifecycle signaling. Vhost emits 'connect', 'disconnect', 'paused'; Subscription emits 'message', 'error', 'subscribed'.

3. **Topology-as-Code:** Configuration is deeply nested JSON describing the entire RabbitMQ topology (vhosts, exchanges, queues, bindings). Rascal asserts this topology on broker connect, enabling declarative, repeatable deployments.

4. **Task Composition Pattern:** async.compose chains curried tasks into pipelines, accumulating context as it flows through. This enables modular, testable initialization logic.

5. **Connection Resilience:** Exponential backoff with configurable min/max for reconnection; connection URL cycling via async.retry in tasks; lazy subscription (defers channel.consume until message listener attached).

6. **Recovery Strategies:** Multi-level, configurable recovery for failed message processing: ack, nack, forward, dlq, with deferral and fallback support. SubscriberError.js orchestrates strategy execution.

7. **Channel Pooling:** Each vhost maintains two generic-pool instances for regular and confirm (publisher-confirms) channels, enabling efficient channel reuse and backpressure handling.

8. **No Database Layer:** Rascal is a pure messaging client library with no persistence layer. State is in-memory (channel pools, subscriber sessions) and ephemeral.

9. **Management API Integration:** Lightweight HTTP client for the RabbitMQ management REST API (vhost CRUD), enabling topology assertion without raw AMQP declarations.

10. **Namespacing & Multi-Tenancy:** Fully-qualified naming (FQN) pattern enables namespace prefixes and uniqueness suffixes on all topology objects, supporting test isolation and multi-tenant deployments.
