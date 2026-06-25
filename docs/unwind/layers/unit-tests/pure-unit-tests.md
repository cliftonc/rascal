# Pure Unit Tests

True unit tests — no broker, synchronous (backoff) or `async`-driven (counter) — exercising small `lib` modules directly.

## exponential.tests.js [MUST] <!-- id: file:test/backoff/exponential.tests.js:exponential.tests.js -->

[test/backoff/exponential.tests.js](https://github.com/cliftonc/rascal/blob/master/test/backoff/exponential.tests.js#L1-L69)

Module under test: `require('../../lib/backoff/exponential')`. Builds a backoff via `exponential({ min, factor, max, randomise })` and calls `.next()` / `.reset()`.

- should backoff by 1 seconds by default — default `next()` sequence 1000, 2000, 4000.
- should backoff by the specified value — `{ min: 2000, factor: 3 }` -> 2000, 6000, 18000.
- should backoff between the specified values — with `randomise: true`, each `next()` falls within `[base, base*factor]` for successive bases (asserts ranges for 7 iterations).
- should cap values — with `max: 18000`, values beyond the cap stay at 18000 (700 iterations).
- should reset values — after `.reset()`, sequence restarts at the first range.

## linear.tests.js [MUST] <!-- id: file:test/backoff/linear.tests.js:linear.tests.js -->

[test/backoff/linear.tests.js](https://github.com/cliftonc/rascal/blob/master/test/backoff/linear.tests.js#L1-L28)

Module under test: `require('../../lib/backoff/linear')`.

- should backoff by 1 seconds by default — `next()` always 1000.
- should backoff by the specified value — `{ min: 2000 }` -> always 2000.
- should backoff by within a range — `{ min: 1000, max: 1002 }` over 1000 iterations yields exactly `[1000, 1001, 1002]`.

## inMemory.tests.js [MUST] <!-- id: file:test/caches/inMemory.tests.js:inMemory.tests.js -->

[test/caches/inMemory.tests.js](https://github.com/cliftonc/rascal/blob/master/test/caches/inMemory.tests.js#L1-L50)

Module under test: `require('../../lib/counters/inMemory')` (the redelivery counter). `beforeEach` creates `inMemory({ size: 3 })`. Uses `async.eachSeries` and the `(test, done)` signature.

- should return increment and get entries — `incrementAndGet` returns 2 for the second occurrence of `'one'`, 1 for `'two'`.
- should limit the counter size — an LRU of size 3 evicts the oldest key; after 4 distinct keys, re-incrementing the evicted `'one'` returns 1 (count reset).
