# API Layer

Rascal is a config-driven RabbitMQ (amqplib) wrapper library. There is **no HTTP server**. The "API" surface consists of:

1. **Public exports** — the object returned by `require('rascal')` ([public-exports.md](public-exports.md)).
2. **RabbitMQ Management HTTP client** — an internal REST client used to assert/check/delete vhosts ([management-client.md](management-client.md)).

## Sections
- [Public Exports](public-exports.md) — the `require('rascal')` contract
- [Management Client](management-client.md) — RabbitMQ Management REST client

## Public Export Summary

| Export | Kind | Tag | Source |
|--------|------|-----|--------|
| `Broker` | Constructor / class | MUST | [index.js#L4,L10](https://github.com/cliftonc/rascal/blob/master/index.js#L4-L10) |
| `BrokerAsPromised` | Constructor / class | MUST | [index.js#L5,L11](https://github.com/cliftonc/rascal/blob/master/index.js#L5-L11) |
| `createBroker(config, [components], callback)` | Factory (callback) | MUST | [index.js#L12](https://github.com/cliftonc/rascal/blob/master/index.js#L12-L12) |
| `createBrokerAsPromised(config, [components])` | Factory (promise) | MUST | [index.js#L13](https://github.com/cliftonc/rascal/blob/master/index.js#L13-L13) |
| `defaultConfig` | Config object | MUST | [index.js#L2,L14](https://github.com/cliftonc/rascal/blob/master/index.js#L2-L14) |
| `testConfig` | Config object | MUST | [index.js#L3,L15](https://github.com/cliftonc/rascal/blob/master/index.js#L3-L15) |
| `withDefaultConfig(config)` | Function | MUST | [index.js#L16-L18](https://github.com/cliftonc/rascal/blob/master/index.js#L16-L18) |
| `withTestConfig(config)` | Function | MUST | [index.js#L19-L21](https://github.com/cliftonc/rascal/blob/master/index.js#L19-L21) |
| `counters` | Object `{stub, inMemory, inMemoryCluster}` | MUST | [index.js#L6,L22](https://github.com/cliftonc/rascal/blob/master/index.js#L6-L22) |

## Management Client Summary

| Method | HTTP | Path | Source |
|--------|------|------|--------|
| `assertVhost(name, config, next)` | PUT | `{url}/api/vhosts/{name}` | [Client.js#L8-L16](https://github.com/cliftonc/rascal/blob/master/lib/management/Client.js#L8-L16) |
| `checkVhost(name, config, next)` | GET | `{url}/api/vhosts/{name}` | [Client.js#L18-L26](https://github.com/cliftonc/rascal/blob/master/lib/management/Client.js#L18-L26) |
| `deleteVhost(name, config, next)` | DELETE | `{url}/api/vhosts/{name}` | [Client.js#L28-L36](https://github.com/cliftonc/rascal/blob/master/lib/management/Client.js#L28-L36) |
| `_request(method, url, options, next)` | (internal) | — | [Client.js#L38-L49](https://github.com/cliftonc/rascal/blob/master/lib/management/Client.js#L38-L49) |
