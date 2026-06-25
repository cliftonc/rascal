# Domain Model

The "domain model" of rascal is **configuration-centric**. There are no runtime
entity classes; instead the domain is a set of plain-object **config entities**
(vhost, connection, exchange, queue, binding, publication, subscription, shovel,
redelivery counter, encryption profile) whose shapes are defined by layered
**default presets**, whose names are computed by **fully-qualified-naming**
rules, whose raw input is **normalized**, and whose invariants are enforced by
**validation**.

These files run as a pipeline: `defaults`/`baseline` supply entity shapes →
`configure.js` normalizes raw user config and computes FQNs → `validate.js`
enforces invariants. `fqn.js` is the naming helper; `tests.js`/`defaults.js` are
the default presets.

## Sections

- [Entities](entities.md) — config entity shapes and their default values (`baseline.js`) — 11 entities
- [Value Objects](value-objects.md) — fully-qualified naming + default presets (`fqn.js`, `defaults.js`, `tests.js`)
- [Enums](enums.md) — 7 closed value sets (connection strategy, retry strategy, destination type, exchange type, counter type, pool name, protocol)
- [Validation](validation.md) — normalization (`configure.js`) and invariants (`validate.js`), plus the `schema.json` mirror

## Seed Coverage

All 6 seeded items documented; 1 extra item added (`schema.json`).

| Seed id | Documented in |
|---------|---------------|
| `file:lib/config/baseline.js:baseline.js` | entities.md |
| `file:lib/config/configure.js:configure.js` | validation.md |
| `file:lib/config/defaults.js:defaults.js` | value-objects.md |
| `file:lib/config/fqn.js:fqn.js` | value-objects.md |
| `file:lib/config/tests.js:tests.js` | value-objects.md |
| `file:lib/config/validate.js:validate.js` | validation.md |
| `file:lib/config/schema.json:schema.json` (added) | validation.md |

## Summary

The domain is a config-driven RabbitMQ topology with **11 entity types**: Vhost,
Connection, PublicationChannelPool, Exchange, Queue, Binding, Publication,
Subscription, RedeliveryCounter, Shovel, and EncryptionProfile. Entity shapes and
defaults come from `baseline.js`, layered under the `defaults.js` (production) and
`tests.js` (test) presets via `_.defaultsDeep`.

Key invariants: every config needs ≥1 vhost; bindings/publications/subscriptions
must reference existing exchanges/queues within their vhost; a publication targets
an exchange XOR a queue; shovels reference an existing subscription+publication;
encryption profiles require key/algorithm/ivLength; connectionStrategy ∈
{random, fixed}; channel pools ⊆ {regularPool, confirmPool}; and validation
allow-lists every entity's attributes. Naming is identity-bearing: `fqn.qualify`
produces `[namespace:]name[:unique]`, namespace `true` and queue `replyTo` true
become random UUIDs, and exchange/queue FQNs flow into publication `destination`
and subscription `source`.
