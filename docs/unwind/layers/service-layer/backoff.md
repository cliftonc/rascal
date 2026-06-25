# Backoff Strategies

Retry-delay timer factory used by `Subscription` (resubscription) and `Vhost`
(reconnection). The factory selects one of two strategies and returns a
`{ next, reset }` timer.

## Public Contract [MUST]

```js
const timer = backoff(options);
timer.next();   // -> number (delay in ms)
timer.reset();  // -> void (resets to starting delay; no-op for linear)
```

---

### index.js [MUST] <!-- id: file:lib/backoff/index.js:index.js -->

[lib/backoff/index.js#L1-L13](https://github.com/cliftonc/rascal/blob/master/lib/backoff/index.js#L1-L13)

Strategy-selector factory. Despite the candidate note ("reset/pause/resume"),
the file contains no pause/resume logic — it only dispatches to a strategy.

```js
const exponential = require('./exponential');
const linear = require('./linear');

const strategies = {
  exponential,
  linear,
};

module.exports = function (options) {
  if (options.delay) return strategies.linear({ min: options.delay });
  if (options.strategy) return strategies[options.strategy](options);
  return strategies.exponential();
};
```

**Selection chain [MUST]:**

1. If `options.delay` is truthy → `linear({ min: options.delay })` (fixed delay, since linear `max` defaults to `min`).
2. Else if `options.strategy` is set → `strategies[options.strategy](options)` (`'exponential'` or `'linear'`).
3. Else → `exponential()` with no options (all defaults).

`options` is the subscription/connection `retry` config object. `options` is
assumed truthy (callers pass `{}` or a config object).

---

### exponential.js [MUST] <!-- id: file:lib/backoff/exponential.js:exponential.js -->

[lib/backoff/exponential.js#L1-L27](https://github.com/cliftonc/rascal/blob/master/lib/backoff/exponential.js#L1-L27)

Exponential backoff with optional jitter. State variable `lower` grows by
`factor` on each `next()` call until it exceeds `max`, after which `max` is
returned permanently (until `reset`).

```js
const get = require('lodash').get;

module.exports = function (options) {
  const min = get(options, 'min', 1000);
  const max = get(options, 'max', Math.pow(min, 10));
  const factor = get(options, 'factor', 2);
  const randomise = get(options, 'randomise', true);
  let lower = min;

  function next() {
    if (lower > max) return max;
    const upper = lower * factor;
    const value = randomise ? Math.floor(Math.random() * (upper - lower + 1) + lower) : lower;
    const capped = Math.min(max, value);
    lower = upper;
    return capped;
  }

  function reset() {
    lower = min;
  }

  return {
    next,
    reset,
  };
};
```

**Constants / defaults [MUST]:**

| Option | Default | Location |
|--------|---------|----------|
| min | 1000 (ms) | exponential.js:4 |
| max | `Math.pow(min, 10)` | exponential.js:5 |
| factor | 2 | exponential.js:6 |
| randomise | true | exponential.js:7 |

**`next()` formula [MUST]:**

```
if (lower > max) return max
upper  = lower * factor
value  = randomise ? floor(random() * (upper - lower + 1) + lower) : lower
capped = min(max, value)
lower  = upper          // advance state for next call
return capped
```

**Edge cases [MUST]:**

| Condition | Behavior |
|-----------|----------|
| `randomise: true` | `value` is a uniform integer in `[lower, upper]` (jittered) |
| `randomise: false` | `value === lower` (deterministic geometric sequence) |
| `lower > max` | returns `max`, state no longer advances |
| `value > max` | capped to `max` |

**Default max note [MUST]:** with default `min = 1000`, `max = 1000^10 = 1e30`,
effectively uncapped. `reset()` sets `lower = min`.

---

### linear.js [MUST] <!-- id: file:lib/backoff/linear.js:linear.js -->

[lib/backoff/linear.js#L1-L11](https://github.com/cliftonc/rascal/blob/master/lib/backoff/linear.js#L1-L11)

Stateless uniform-random delay between `min` and `max`. Despite the name, it
does not increase per attempt — each `next()` is an independent draw. `reset` is
a no-op.

```js
module.exports = function (options = {}) {
  const min = options.min || 1000;
  const max = options.max || min;
  const range = max - min;

  function next() {
    return range === 0 ? min : Math.floor(Math.random() * (range + 1) + min);
  }

  return { next, reset: () => {} };
};
```

**Constants / defaults [MUST]:**

| Option | Default | Location |
|--------|---------|----------|
| min | 1000 (ms) | linear.js:2 |
| max | `min` (i.e. fixed delay) | linear.js:3 |

**`next()` formula [MUST]:**

```
range = max - min
if range === 0 -> return min                 // fixed delay
else           -> floor(random() * (range+1) + min)   // uniform int in [min, max]
```

When invoked via `index.js` with `options.delay`, only `min` is set, so
`max === min`, `range === 0`, and `next()` always returns the fixed `delay`.
`reset()` does nothing (state-free).
