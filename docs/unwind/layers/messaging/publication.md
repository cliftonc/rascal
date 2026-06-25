# Publication

Outbound message abstraction. Borrows a pooled channel from the vhost, publishes/sends with optional confirms and encryption, and returns a `PublicationSession` event emitter for the async result.

## Publication.js [MUST] <!-- id: file:lib/amqp/Publication.js:Publication.js -->

[Publication.js](https://github.com/cliftonc/rascal/blob/master/lib/amqp/Publication.js#L1-L272)

### Factory: `Publication.create(vhost, config, next)` [MUST]

Selects the publish function and channel-pool (regular vs confirm) by config shape:

```js
if (hasOwnProperty(config, 'exchange') && config.confirm) → publishToConfirmExchange, confirm channel pool
if (hasOwnProperty(config, 'exchange'))                   → publishToExchange,        regular channel pool
if (config.queue && config.confirm)                       → sendToConfirmQueue,       confirm channel pool
if (config.queue)                                          → sendToQueue,              regular channel pool
```

The matching `borrow*/return*/destroy*` channel functions are bound from the vhost and passed into the `Publication`. `init` simply yields `next(null, self)`.

### publish(payload, overrides, next) [MUST]

```js
this.publish = function (payload, overrides, next) {
  const publishConfig = _.defaultsDeep({}, overrides, config);
  const content = getContent(payload);
  publishConfig.options.contentType = publishConfig.options.contentType || content.type;
  publishConfig.options.messageId   = publishConfig.options.messageId   || uuid();
  publishConfig.options.replyTo     = publishConfig.options.replyTo     || publishConfig.replyTo;
  publishConfig.encryption ? _publishEncrypted(content.buffer, publishConfig, next) : _publish(content.buffer, publishConfig, next);
};
```

**Content negotiation (`getContent`):**
- `Buffer` → `{ buffer: payload, type: undefined }`
- `string` → `{ buffer: Buffer.from(payload), type: 'text/plain' }`
- else → `{ buffer: Buffer.from(JSON.stringify(payload)), type: 'application/json' }`

`messageId` defaults to a uuid; `contentType` defaults to the negotiated type.

### forward(message, overrides, next) [MUST]

Re-publishes a consumed message (used by shovels and the `forward` recovery strategy).

```js
const originalQueue = message.properties.headers.rascal.originalQueue;
const publishConfig = _.defaultsDeep({}, overrides, config, { routingKey: message.fields.routingKey });
publishConfig.options = _.defaultsDeep(publishConfig.options, message.properties);
_.set(publishConfig, 'options.headers.rascal.restoreRoutingHeaders', !!publishConfig.restoreRoutingHeaders);
_.set(publishConfig, 'options.headers.rascal.originalExchange', message.fields.exchange);
_.set(publishConfig, 'options.headers.rascal.originalRoutingKey', message.fields.routingKey);
_.set(publishConfig, 'options.CC', _.chain([]).concat(publishConfig.options.CC, format('%s.%s', originalQueue, publishConfig.routingKey)).uniq().compact().value());
_publish(message.content, publishConfig, next);
```

Records original exchange/routingKey in `headers.rascal`, and appends `<originalQueue>.<routingKey>` to the `CC` list (de-duplicated).

### Encryption [MUST]

`_publishEncrypted` calls `encrypt(algorithm, key, ivLength, buffer, cb)`:

```js
crypto.randomBytes(ivLength) → crypto.createCipheriv(algorithm, Buffer.from(keyHex,'hex'), iv)
// cipher.update + cipher.final → encrypted buffer; iv returned as hex
```

Then sets headers and content type:
```js
options.headers.rascal.encryption.name = encryptionConfig.name
options.headers.rascal.encryption.iv   = iv (hex)
options.headers.rascal.encryption.originalContentType = <prior contentType>
options.contentType = 'application/octet-stream'
```

(Decryption is the mirror in `Subscription.getContent` — see [subscription.md](subscription.md).)

### Core publish flow `_publish(buffer, publishConfig, next)` [MUST]

```js
const session = new PublicationSession(vhost, messageId);
borrowChannelFn((err, channel) => {
  session._removePausedListener();
  if (err) return session.emit('error', err, messageId);
  if (session.isAborted()) return abortPublish(channel, messageId);    // return channel, no publish
  const disconnectionHandler = makeDisconnectionHandler(channel, messageId, session, config);
  const returnHandler = session.emit.bind(session, 'return');
  addListeners(channel, disconnectionHandler, returnHandler);          // error/return on channel; error/close on connection
  try {
    session._startPublish();
    publishFn(channel, buffer, publishConfig, (err, ok) => {
      session._endPublish();
      if (err) { destroyChannel(channel, ...); return session.emit('error', err, messageId); }
      ok ? returnChannel(channel, ...) : deferReturnChannel(channel, ...); // ok=false => wait for 'drain'
      session.emit('success', messageId);
    });
  } catch (err) { returnChannel(channel, ...); return session.emit('error', err, messageId); }
});
next(null, session);   // session returned synchronously; result delivered via events
```

`next(null, session)` returns immediately; the publish result is delivered asynchronously through session events. `deferReturnChannel` returns the channel only after the channel emits `drain` (back-pressure handling).

### Publish functions: confirm vs no-confirm [MUST]

```js
publishToExchange       → channel.publish(destination, routingKey, content, options)        via publishNoConfirm
publishToConfirmExchange → channel.publish(destination, routingKey, content, options, cb)    via publishAndConfirm
sendToQueue             → channel.sendToQueue(destination, content, options)                 via publishNoConfirm
sendToConfirmQueue      → channel.sendToQueue(destination, content, options, cb)             via publishAndConfirm
```

- **`publishNoConfirm(fn, channel, next)`** — listens once for `drain`; `ok = fn()`; `next(null, ok || drained)`. (`ok` is amqplib's back-pressure boolean.)
- **`publishAndConfirm(fn, channel, config, next)`** — wraps `next` in `_.once`; if `config.timeout`, arms `setConfirmationTimeout` (rejects with "Timedout after Nms waiting for broker to confirm…"); listens for `drain`; `fn(cb)` invokes cb on broker confirm, clearing the timeout, `next(err, ok || drained)`.

### Channel listener management [MUST]

- `addListeners` — `channel.on('error', disconnectionHandler)`, `channel.on('return', returnHandler)`, `channel.connection.once('error'|'close', disconnectionHandler)`.
- `removeListeners` — removes `drain`, `error`, `return` from channel and `error`/`close` from connection.
- `makeDisconnectionHandler` — `_.once`, `setImmediate`-wrapped; close-with-error → `handleChannelError` (emits `error`); plain close → `handleChannelClose` (emits `close`).
- `returnChannel` removes listeners then returns to pool; `destroyChannel` removes listeners then destroys.

### Events emitted (on PublicationSession) [MUST]

| Event | Payload | When |
|-------|---------|------|
| `success` | `messageId` | publish/confirm completed |
| `error` | `(err, messageId)` | borrow failure, publish throw, confirm error, channel error |
| `return` | amqplib return args | mandatory/immediate message returned by broker |
| `close` | `messageId` | channel closed during publication |
| `paused` | `messageId` | vhost is paused (see PublicationSession) |

---

## PublicationSession.js [MUST] <!-- id: file:lib/amqp/PublicationSession.js:PublicationSession.js -->

[PublicationSession.js](https://github.com/cliftonc/rascal/blob/master/lib/amqp/PublicationSession.js#L1-L45) · `inherits(PublicationSession, EventEmitter)`

Per-publish result emitter, returned synchronously from `publish`/`forward`.

```js
function PublicationSession(vhost, messageId) {
  this.stats = {};
  this.abort = () => { aborted = true; };               // set before borrow completes to skip the publish
  this.isAborted = () => aborted;
  this._removePausedListener = () => vhost.removeListener('paused', emitPaused);
  this._startPublish = () => { this.started = Date.now(); };
  this._endPublish   = () => { this.stats.duration = Date.now() - this.started; };
  function emitPaused() { self.emit('paused', messageId); }
  vhost.on('paused', emitPaused);                        // forward vhost pause to the publisher
  self.on('newListener', (event) => { if (event === 'paused' && vhost.isPaused()) emitPaused(); }); // late-listener catch-up
}
```

- `abort()` lets a caller cancel a publish before the channel is borrowed.
- While waiting for a channel, the session relays the vhost's `paused` event so a caller can react to a paused vhost; the listener is removed once the channel is borrowed (`_removePausedListener`).
- The `newListener` hook emits `paused` immediately if a listener is added while the vhost is already paused.
- `stats.duration` records publish duration.
