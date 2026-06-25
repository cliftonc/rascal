# Management Client (RabbitMQ Management REST API)

An internal HTTP client that talks to the [RabbitMQ Management plugin's REST API](https://www.rabbitmq.com/management.html) to manage vhosts. It is **not** part of the public `require('rascal')` export — it is consumed internally by the broker init/teardown tasks. It is documented here because it is the only outbound HTTP contract in the library.

Source: [lib/management/Client.js#L1-L56](https://github.com/cliftonc/rascal/blob/master/lib/management/Client.js#L1-L56)

```js
const http = require('http');
const debug = require('debug')('rascal:management:client');
const format = require('util').format;

function Client(agent) {
  const self = this;

  this.assertVhost = function (name, config, next) { ... };
  this.checkVhost  = function (name, config, next) { ... };
  this.deleteVhost = function (name, config, next) { ... };
  this._request    = function (method, url, options, next) { ... };
  function getUrl(name, config) { ... }
}

module.exports = Client;
```

## Constructor

`new Client(agent)` — `agent` is an `http.Agent` (passed through from `ctx.components.agent`, see [assertVhost.js#L10](https://github.com/cliftonc/rascal/blob/master/lib/amqp/tasks/assertVhost.js#L10-L10)). It is forwarded into every `http.request` call so connection pooling/keep-alive is shared.

## URL construction

Source: [Client.js#L51-L53](https://github.com/cliftonc/rascal/blob/master/lib/management/Client.js#L51-L53)

```js
function getUrl(name, config) {
  return format('%s/%s/%s', config.url, 'api/vhosts', name);
}
```

The request URL is `` `${config.url}/api/vhosts/${name}` ``. `config` is the per-connection **management** config object (`connection.management`), built during configuration.

### How `config` (the management connection) is built

Source: [lib/config/configure.js#L108-L136](https://github.com/cliftonc/rascal/blob/master/lib/config/configure.js#L108-L136)

```js
function configureManagementConnection(vhostConfig, vhostName, connection) {
  connection.management = _.isString(connection.management) ? { url: connection.management } : connection.management;
  const attributesFromUrl = parseConnectionUrl(connection.management.url);
  const attributesFromConfig = getConnectionAttributes(connection.management);
  const defaults = { user: connection.user, password: connection.password, hostname: connection.hostname };

  const connectionAttributes = _.defaultsDeep({ options: null }, attributesFromUrl, attributesFromConfig, defaults);
  setConnectionAttributes(connection.management, connectionAttributes);
  setConnectionUrls(connection.management);
}

function setConnectionUrls(connection) {
  const auth = getAuth(connection.user, connection.password);
  const pathname = connection.vhost === '/' ? '' : connection.vhost;
  const query = connection.options;
  connection.url = url.format({ slashes: true, ...connection, auth, pathname, query });
  connection.loggableUrl = connection.url.replace(/:[^:]*@/, ':***@');
}

function getAuth(user, password) {
  return user && password ? `${user}:${password}` : undefined;
}
```

Key fields on the management `config` consumed by `Client`:
- **`config.url`** — the base URL of the RabbitMQ Management API (e.g. `http://user:pass@host:15672`). The user/password are embedded in the URL's auth component (HTTP Basic auth, via Node's `http.request` parsing of the URL credentials).
- **`config.loggableUrl`** — the same URL with the password masked (`:***@`); used only in error messages so credentials are never logged ([configure.js#L131](https://github.com/cliftonc/rascal/blob/master/lib/config/configure.js#L131-L131)).
- **`config.options`** — extra `http.request` options spread into each request (`{ ...options, method, agent }`).

**Authentication:** HTTP Basic, carried inside `config.url` as `user:password@`. Derived from the management connection's own user/password, falling back to the AMQP connection's `user`/`password`/`hostname` ([configure.js#L112](https://github.com/cliftonc/rascal/blob/master/lib/config/configure.js#L112-L112)).

## Methods

### assertVhost(name, config, next) [MUST] <!-- id: export:lib/management/Client.js:assertVhost -->

Source: [Client.js#L8-L16](https://github.com/cliftonc/rascal/blob/master/lib/management/Client.js#L8-L16)

```js
this.assertVhost = function (name, config, next) {
  debug('Asserting vhost: %s', name);
  const url = getUrl(name, config);
  self._request('PUT', url, config.options, (err) => {
    if (!err) return next();
    const _err = err.status ? new Error(format('Failed to assert vhost: %s. %s returned status %d', name, config.loggableUrl, err.status)) : err;
    return next(_err);
  });
};
```

- **HTTP:** `PUT {config.url}/api/vhosts/{name}` — idempotently creates the vhost (RabbitMQ semantics).
- **Success:** calls `next()` with no error.
- **Failure:** if the error has a `status` (HTTP >= 300), wraps it as `Error('Failed to assert vhost: {name}. {loggableUrl} returned status {status}')`; otherwise (transport error) passes the raw error through.

### checkVhost(name, config, next) [MUST] <!-- id: export:lib/management/Client.js:checkVhost -->

Source: [Client.js#L18-L26](https://github.com/cliftonc/rascal/blob/master/lib/management/Client.js#L18-L26)

```js
this.checkVhost = function (name, config, next) {
  debug('Checking vhost: %s', name);
  const url = getUrl(name, config);
  self._request('GET', url, config.options, (err) => {
    if (!err) return next();
    const _err = err.status ? new Error(format('Failed to check vhost: %s. %s returned status %d', name, config.loggableUrl, err.status)) : err;
    return next(_err);
  });
};
```

- **HTTP:** `GET {config.url}/api/vhosts/{name}` — verifies the vhost exists.
- **Success:** `next()`. **Failure:** same wrapping pattern as `assertVhost` ("Failed to check vhost: ...").

### deleteVhost(name, config, next) [MUST] <!-- id: export:lib/management/Client.js:deleteVhost -->

Source: [Client.js#L28-L36](https://github.com/cliftonc/rascal/blob/master/lib/management/Client.js#L28-L36)

```js
this.deleteVhost = function (name, config, next) {
  debug('Deleting vhost: %s', name);
  const url = getUrl(name, config);
  self._request('DELETE', url, config.options, (err) => {
    if (!err) return next();
    const _err = err.status ? new Error(format('Failed to delete vhost: %s. %s returned status %d', name, config.loggableUrl, err.status)) : err;
    return next(_err);
  });
};
```

- **HTTP:** `DELETE {config.url}/api/vhosts/{name}` — removes the vhost.
- **Success:** `next()`. **Failure:** same wrapping pattern ("Failed to delete vhost: ...").

### _request(method, url, options, next) [SHOULD] <!-- id: export:lib/management/Client.js:_request -->

Internal HTTP transport shared by all three public methods.

Source: [Client.js#L38-L49](https://github.com/cliftonc/rascal/blob/master/lib/management/Client.js#L38-L49)

```js
this._request = function (method, url, options, next) {
  const req = http.request(url, { ...options, method, agent }, (res) => {
    if (res.statusCode >= 300) {
      const err = Object.assign(new Error('HTTP Error'), { status: res.statusCode });
      return next(err);
    }
    res.on('data', () => {});
    res.on('end', () => next());
  });
  req.on('error', next);
  req.end();
};
```

Contract / semantics:
- Issues `http.request(url, { ...config.options, method, agent })` with an empty body (`req.end()` with no payload).
- **HTTP status >= 300 = failure:** any response status code of 300 or greater produces an `Error('HTTP Error')` decorated with `.status = res.statusCode`. This `.status` is what the wrapping methods test for to build their descriptive error messages.
- Status codes < 300 are treated as success: the response body is drained (`res.on('data', () => {})`) and `next()` is called with no error on `end`.
- Transport-level errors (`req.on('error', next)`) propagate the raw error (no `.status`), so callers pass them through unchanged.

## Retry / URL-cycling (caller side)

The `Client` itself performs no retries. URL-cycling across multiple connection candidates is handled by the broker tasks that invoke it.

Source: [lib/amqp/tasks/assertVhost.js#L6-L29](https://github.com/cliftonc/rascal/blob/master/lib/amqp/tasks/assertVhost.js#L6-L29) (the `checkVhost` and `deleteVhost` tasks follow the same pattern)

```js
module.exports = _.curry((config, ctx, next) => {
  if (!config.assert) return next(null, config, ctx);

  const candidates = config.connections;
  const client = new Client(ctx.components.agent);

  async.retry(
    candidates.length,
    (cb) => {
      const connectionConfig = candidates[ctx.connectionIndex];
      client.assertVhost(config.name, connectionConfig.management, (err) => {
        if (err) {
          ctx.connectionIndex = (ctx.connectionIndex + 1) % candidates.length;
          return cb(err);
        }
        ctx.connectionConfig = connectionConfig;
        cb();
      });
    },
    (err) => { next(err, config, ctx); },
  );
});
```

- Uses `async.retry` with the retry count equal to the number of connection `candidates`.
- On each attempt it picks `candidates[ctx.connectionIndex].management` as the `config` for the client.
- On failure it advances `ctx.connectionIndex` round-robin (`(index + 1) % candidates.length`) and retries against the next connection's management endpoint.
- On success it records the working `connectionConfig` on `ctx`.
- Gated by `config.assert` (assertVhost task) — skipped entirely when assertion is disabled.
