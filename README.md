# subservice
Multi-language service configuration and integration testing.

## Javascript

Imagine a Node app in `PROJECT_DIR/node-auth/`.

```js
// PROJECT_DIR/node-auth/src/index.js

subservice.load('../subsurface.yml')
  .then(({mysqlClient, amqpClient}) => initApp({mysqlClient, amqpClient}))
  .then(({app}) => app.start())
  .catch(err => logger.error(err))
```

```bash
# in PROJECT_DIR/node-auth/
subsurface up

# read from PROJECT_DIR/node-auth/subsurface.yml
# build Docker containers for node, mysql, and rabbitmq
# run Docker containers for mysql, rabbitmq
# create MYSQL_URL, AMQP_URL env variables
# run node app in Docker container with fully configured environment
```

```yml
# PROJECT_DIR/node-auth/subsurface.yml

node: node src/index.js
mysql: mysqlClient
rabbitmq: amqpClient
```

```js
# subsurface.js

const loadYML = () => new Promise((resolve, reject) => {
  // ...
})

const load = (ymlFilePath) => new Promise((resolve, reject) => {
  loadYML(ymlFilePath)
    .then(({services}) => {
      // .. get MYSQL_URL, AMQP_URL env variables
      let nodeServices = {}
      // ... dynamically build mysqlClient and amqpClient and add to nodeServices obj
      resolve(nodeServices)
    })
})
```

## Ruby

Imagine a Rails app in `PROJECT_DIR/rails-api/`.

```ruby
# PROJECT_DIR/rails-api/Gemfile

gem 'activerecord-subsurface-adapter', '~> 1.2.3'
gem 'subsurface', '~> 1.2.3'
```

```yml
# PROJECT_DIR/rails-api/config/database.yml
platform: &platform
  adapter: subsurface
  
development:
  <<: *default
  
test:
  <<: *default
  
production:
  <<: *default
  
# adapter reads from PROJECT_DIR/rails-api/subsurface.yml and builds rails adapters
# gets host info from MYSQL_URL, AMQP_URL env variables
```

```yml
# PROJECT_DIR/api/subsurface.yml

rails: rails s
mysql: activerecord-mysql-adapter
rabbitmq: amqp
```

```bash
# in PROJECT_DIR/rails-api
subsurface up

# builds Docker containers for rails, mysql and rabbitmq
# run Docker containers for mysql and rabbitmq
# create MYSQL_URL, AMQP_URL env variables
# run Rails app in Docker container with fully configured environment
```

## In production

Bypass Docker creation and inject parent env variables instead:

```bash
# in PROJECT_DIR/rails-api/
subservice up --rabbitmq AMQP_URL --mysql MYSQL_URL
```

Get services and env variables from a discovery service like etcd, Consul or Zookeeper

```bash
# in PROJECT_DIR/node-auth/
subservice up --discovery

# looks up service information for mysql and rabbitmq
```

# Integration tests

Subservice supports nested services via nested folders with `subservice.yml`

```
# PROJECT_DIR/subservice.yml
test:
  rabbitmq: amqpClient
  auth: node-auth/subservice.yml
  search: rails-search/subservice.yml
  api: rails-api/subservice.yml
  command: node test/integration-spec.js | tap-spec # or rspec
```

```bash
# in PROJECT_DIR/
subservice test

# builds and runs rabbitmq
# passes in AMQP_URL to node-auth, rails-search, rails-api
# node-auth, rails-search, rails-api launch their own test services and setup their own databases
# runs command node test/integration-spec.js pipes resuls to tap-spec
```

Javascript test, although could be written in any language:

```js
# test/integration-spec.js

test('integration tests', t => {
  t.test('should get health check from auth service', t => {
    get('auth/healthcheck')
      .then({res} => t.equal(res, 'success'))
      .then(t.end)
  })
  
  t.test('should post new gizmo to api service and see updates in the search service', t => {
    post('api/gizmo', {name: 'fizzbuzz', color: 'red'})
      .then(res => t.equal(res, 'success'))
      .then(() => get('search?type=gizmo&name=fizzbuzz'))
      .then(res => t.equal(res.color, 'red'))
      .then(t.end)
  })
})
```
