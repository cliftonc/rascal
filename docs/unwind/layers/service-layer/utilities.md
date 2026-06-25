# Utilities

Shared timer helper used across the service/amqp layer for scheduling without
keeping the Node.js event loop alive.

---

### setTimeoutUnref.js [SHOULD] <!-- id: file:lib/utils/setTimeoutUnref.js:setTimeoutUnref.js -->

[lib/utils/setTimeoutUnref.js#L1-L5](https://github.com/cliftonc/rascal/blob/master/lib/utils/setTimeoutUnref.js#L1-L5)

Wraps `setTimeout` and calls `.unref()` on the returned timer so a pending
timeout does not prevent the process from exiting. Falls back to returning the
timer object directly in environments where `unref` is unavailable (browser).
Tagged [SHOULD] — generic timer helper, not business logic.

```js
// See https://github.com/onebeyond/rascal/issues/89
module.exports = function (fn, millis) {
  const t = setTimeout(fn, millis);
  return t.unref ? t.unref() : t;
};
```

**Contract [SHOULD]:**

```js
setTimeoutUnref(fn, millis)
```

- Schedules `fn` after `millis` milliseconds.
- If the timer object exposes `unref` (Node.js), calls `t.unref()` and returns its
  result.
- Otherwise returns the raw timer `t`.

**Consumers:**
- `SubscriberError.js:19` — retry scheduling.
- `Subscription.js:181,198` — error emission and timeout handling.
- `Vhost.js:237,254` — reconnection scheduling.
