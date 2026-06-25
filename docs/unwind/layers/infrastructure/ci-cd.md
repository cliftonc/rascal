# CI / CD

rascal has no container or IaC deployment — it is published to npm. CI runs lint
+ tests against a live RabbitMQ service container; publish is triggered by a
GitHub release.

## Continuous Integration

### node-js-ci.yml [SHOULD] <!-- id: file:.github/workflows/node-js-ci.yml:node-js-ci.yml -->

[node-js-ci.yml](https://github.com/cliftonc/rascal/blob/master/.github/workflows/node-js-ci.yml#L1-L24)

GitHub Actions CI. Trigger: `on: [push]`.

- Single `build` job on `ubuntu-latest`.
- **Service container**: `rabbitmq:3-management-alpine` exposing ports
  `5672:5672` (AMQP) and `15672:15672` (management API). Tests connect to this
  broker.
- **Matrix**: Node `16.x, 18.x, 20.x, 22.x, 24.x`.
- **Steps**: `actions/checkout@v3` → `actions/setup-node@v3` → `npm ci` →
  `npm run lint` → `npm test`.

## Publish / Release

### node-js-publish.yml [MUST] <!-- id: file:.github/workflows/node-js-publish.yml:node-js-publish.yml -->

[node-js-publish.yml](https://github.com/cliftonc/rascal/blob/master/.github/workflows/node-js-publish.yml#L1-L37)

GitHub Actions publish pipeline. Trigger: `on: release: types: [created]`.

- **`build` job**: same RabbitMQ service container (ports 5672/15672), Node
  `16.x`. Runs `npm ci` → `npm run lint` → `npm test`. Gate before publish.
- **`publish-npm` job**: `needs: build`. Checkout → setup-node Node `16.x` with
  `registry-url: https://registry.npmjs.org/` → `npm ci` → `npm publish`.
  Auth via `NODE_AUTH_TOKEN: ${{ secrets.NPM_AUTH_TOKEN }}`.

This is the deployment contract: a GitHub release publishes the package to the
public npm registry. Reproduce in a rebuild by wiring a release-triggered
publish with a registry auth token secret.

## Docs site

### _config.yml [DON'T] <!-- id: file:_config.yml:_config.yml -->

[_config.yml](https://github.com/cliftonc/rascal/blob/master/_config.yml#L1-L1)

GitHub Pages / Jekyll config — single line `theme: jekyll-theme-cayman`. Renders
the README as the project site at the `homepage` URL. Cosmetic; excluded from the
npm tarball via `.npmignore`.
