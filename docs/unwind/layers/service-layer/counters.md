# Counters

Redelivery counters track how many times a given message has been delivered, by
key `"${subscriptionName}/${messageId}"`. `Subscription` calls
`counter.incrementAndGet(key, next)` ([Subscription.js:212](https://github.com/cliftonc/rascal/blob/master/lib/amqp/Subscription.js#L212)) to decide retry/recovery behaviour. `Broker.create` registers the three built-in counters as defaults ([Broker.js:23-27](https://github.com/cliftonc/rascal/blob/master/lib/amqp/Broker.js#L23-L27)).

## Public Contract [MUST]

```js
const counter = counterFactory(options);
counter.incrementAndGet(key, (err, count) => { /* ... */ });
```

- `key`: string identifying the message (`subscriptionName/messageId`).
- callback: `(err, count)` — `count` is the post-increment redelivery count.
- All implementations are error-tolerant: on failure they invoke the callback
  with `(null, 1)` (or `(null, 0)` for stub), never propagating an error.

---

### index.js [SHOULD] <!-- id: file:lib/counters/index.js:index.js -->

[lib/counters/index.js#L1-L9](https://github.com/cliftonc/rascal/blob/master/lib/counters/index.js#L1-L9)

Registry aggregating the three counter factories. Tagged [SHOULD] — it is a
plain module-export barrel with no logic.

```js
const stub = require('./stub');
const inMemory = require('./inMemory');
const inMemoryCluster = require('./inMemoryCluster');

module.exports = {
  stub,
  inMemory,
  inMemoryCluster,
};
```

Note: `inMemoryCluster` exported here is the full `{ master, worker }` object.
`Broker.js:15` imports `require('../counters/inMemoryCluster').worker` specifically.

---

### stub.js [MUST] <!-- id: file:lib/counters/stub.js:stub.js -->

[lib/counters/stub.js#L1-L7](https://github.com/cliftonc/rascal/blob/master/lib/counters/stub.js#L1-L7)

No-op counter; always reports `0` redeliveries. Default counter in baseline
config ([baseline.js:89](https://github.com/cliftonc/rascal/blob/master/lib/config/baseline.js#L89)).

```js
module.exports = function () {
  return {
    incrementAndGet(key, next) {
      next(null, 0);
    },
  };
};
```

**Behavior [MUST]:** ignores `key`, no state, always `next(null, 0)`.

---

### inMemory.js [MUST] <!-- id: file:lib/counters/inMemory.js:inMemory.js -->

[lib/counters/inMemory.js#L1-L15](https://github.com/cliftonc/rascal/blob/master/lib/counters/inMemory.js#L1-L15)

In-process redelivery counter backed by an LRU cache keyed by message key.
Single-process only (counts are not shared across cluster workers).

```js
const _ = require('lodash');
const LRUCache = require('lru-cache');

module.exports = function init(options) {
  const size = _.get(options, 'size') || 1000;
  const cache = new LRUCache({ max: size });

  return {
    incrementAndGet(key, next) {
      const redeliveries = cache.get(key) + 1 || 1;
      cache.set(key, redeliveries);
      next(null, redeliveries);
    },
  };
};
```

**Constants / defaults [MUST]:**

| Option | Default | Location |
|--------|---------|----------|
| size (LRU max entries) | 1000 | inMemory.js:5 |

**Increment formula [MUST]:**

```
redeliveries = cache.get(key) + 1 || 1
```

**Edge cases [MUST]:**

| `cache.get(key)` | `+ 1` | `|| 1` result | count |
|------------------|-------|---------------|-------|
| `undefined` (miss) | `NaN` | falsy → `1` | 1 (first delivery) |
| `1` | `2` | truthy → `2` | 2 |
| `n` | `n+1` | `n+1` | n+1 |

When the LRU evicts a key (more than `size` distinct in-flight messages), that
message's count resets to 1 on its next delivery. Never errors.

---

### inMemoryCluster.js [MUST] <!-- id: file:lib/counters/inMemoryCluster.js:inMemoryCluster.js -->

[lib/counters/inMemoryCluster.js#L1-L71](https://github.com/cliftonc/rascal/blob/master/lib/counters/inMemoryCluster.js#L1-L71)

Cluster-aware counter. The **master** process holds the single authoritative
`inMemory` counter; **worker** processes proxy `incrementAndGet` to the master
over node `cluster` IPC, correlating requests/responses by UUID via `stashback`.
`Broker` uses `.worker` ([Broker.js:15](https://github.com/cliftonc/rascal/blob/master/lib/amqp/Broker.js#L15)); the host app must call `.master(options)` in the primary process.

```js
const cluster = require('cluster');
const uuid = require('uuid').v4;
const Stashback = require('stashback');
const inMemory = require('./inMemory');

const debug = 'rascal:counters:inMemoryCluster';

module.exports = {
  master: function master(options) {
    const workers = {};
    const counter = inMemory(options);

    function handleMessage(worker, message) {
      if (message.sender !== 'rascal-in-memory-cluster-counter' || message.cmd !== 'incrementAndGet') return;
      counter.incrementAndGet(message.key, (err, value) => {
        worker.send({
          sender: 'rascal-in-memory-cluster-counter',
          correlationId: message.correlationId,
          value: err ? 1 : value,
        });
      });
    }

    cluster
      .on('fork', (worker) => {
        workers[worker.id] = worker;
        worker.on('message', (message) => {
          handleMessage(worker, message);
        });
      })
      .on('disconnect', (worker) => {
        delete workers[worker.id];
      })
      .on('exit', (worker) => {
        delete workers[worker.id];
      });
  },
  worker: function worker(options) {
    if (!cluster.isWorker) throw new Error("You cannot use Rascal's in memory cluster counter outside of a cluster");
    if (!options) return worker({});
    const timeout = options.timeout || 100;
    const stashback = Stashback({ timeout });

    process.on('message', (message) => {
      if (message.sender !== 'rascal-in-memory-cluster-counter') return;
      stashback.unstash(message.correlationId, (err, cb) => {
        err ? cb(null, 1) : cb(null, message.value);
      });
    });

    return {
      incrementAndGet(key, cb) {
        const correlationId = uuid();
        stashback.stash(
          correlationId,
          (err, value) => {
            err ? cb(null, 1) : cb(null, value);
          },
          (err) => {
            if (err) return cb(null, 1);
            process.send({
              sender: 'rascal-in-memory-cluster-counter',
              correlationId,
              cmd: 'incrementAndGet',
            });
          },
        );
      },
    };
  },
};
```

**IPC protocol constants [MUST]:**

| Constant | Value | Purpose |
|----------|-------|---------|
| message `sender` | `'rascal-in-memory-cluster-counter'` | namespaces messages; non-matching messages ignored |
| `cmd` | `'incrementAndGet'` | only command supported |
| `correlationId` | `uuid.v4()` | matches worker request to master response |
| `options.timeout` (worker) | 100 (ms) | stashback timeout before falling back to `1` |

**master(options) [MUST]:**
- Creates one shared `inMemory(options)` counter (same `size` default of 1000).
- Tracks live workers in `workers` map (`fork` adds; `disconnect`/`exit` delete).
- On a matching `incrementAndGet` message, increments the shared counter and replies to the originating worker with `{ correlationId, value }` (`value = 1` if the master counter errored).

**worker(options) [MUST]:**
- Throws `"You cannot use Rascal's in memory cluster counter outside of a cluster"` if `!cluster.isWorker`.
- Normalises missing `options` by recursing with `{}`.
- Listens for master replies; `unstash`es the pending callback by `correlationId`.
- `incrementAndGet(key, cb)`: stashes `cb` under a new `correlationId`, then sends the increment request to the master. The `key` travels in the original code path via `message.key` — note the worker's `process.send` does **not** include `key` (see Unknowns).

**Failure fallbacks [MUST]:**

| Failure point | Fallback value |
|---------------|----------------|
| master counter error | `1` (sent as `value`) |
| stashback stash setup error | `cb(null, 1)` |
| stashback timeout / unstash error | `cb(null, 1)` |

## Unknowns

- In `worker.incrementAndGet`, the outbound `process.send(...)` payload
  (inMemoryCluster.js:61-66) does **not** include the `key` field, yet
  `master.handleMessage` reads `message.key` (line 15). As written, the master
  receives `undefined` as the key, so all worker-originated counts share a single
  bucket. This is the verifiable behavior of the current source; intent is not
  documented in code.
