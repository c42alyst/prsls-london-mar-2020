# Module 19: Structured logging

## Apply structured logging to the project

<details>
<summary><b>Using a simple logger</b></summary><p>

I built a suite of tools to help folks build production-ready serverless applications while I was at DAZN. It's now open source: [dazn-lambda-powertools](https://github.com/getndazn/dazn-lambda-powertools).

One of the tools available is a very simple logger that supports structured logging (amongst other things).

So, first, let's install the logger for our project.

1. At the project root, run the command `npm install --save @dazn/lambda-powertools-logger` to install the logger.

Now we need to change all the places where we're using `console.log`.

2. Open `functions/get-index.js`, and replace `console.log` with use of the logger. Replace the entire `get-index.js` with the following.

```javascript
const fs = require("fs")
const Mustache = require('mustache')
const http = require('axios')
const aws4 = require('aws4')
const URL = require('url')
const Log = require('@dazn/lambda-powertools-logger')

const restaurantsApiRoot = process.env.restaurants_api
const days = ['Sunday', 'Monday', 'Tuesday', 'Wednesday', 'Thursday', 'Friday', 'Saturday']
const ordersApiRoot = process.env.orders_api

const awsRegion = process.env.AWS_REGION
const cognitoUserPoolId = process.env.cognito_user_pool_id
const cognitoClientId = process.env.cognito_client_id

let html

function loadHtml () {
  if (!html) {
    Log.info('loading index.html...')
    html = fs.readFileSync('static/index.html', 'utf-8')
    Log.info('loaded')
  }
  
  return html
}

const getRestaurants = async () => {
  Log.debug('getting restaurants...', { url: restaurantsApiRoot })

  const url = URL.parse(restaurantsApiRoot)
  const opts = {
    host: url.hostname,
    path: url.pathname
  }

  aws4.sign(opts)

  const httpReq = http.get(restaurantsApiRoot, {
    headers: opts.headers
  })
  const restaurants = (await httpReq).data
  Log.debug('got restaurants', { count: restaurants.length })

  return restaurants
}

module.exports.handler = async (event, context) => {
  const template = loadHtml()
  const restaurants = await getRestaurants()
  const dayOfWeek = days[new Date().getDay()]
  const view = {
    awsRegion,
    cognitoUserPoolId,
    cognitoClientId,
    dayOfWeek,
    restaurants,
    searchUrl: `${restaurantsApiRoot}/search`,
    placeOrderUrl: `${ordersApiRoot}`
  }
  const html = Mustache.render(template, view)
  const response = {
    statusCode: 200,
    headers: {
      'content-type': 'text/html; charset=UTF-8'
    },
    body: html
  }

  return response
}
```

3. Modify `functions/get-restaurants.js` and add some debug logs. Replace the entire `get-restaurants.js` with the following

```javascript
const AWS = require('aws-sdk')
const dynamodb = new AWS.DynamoDB.DocumentClient()
const Log = require('@dazn/lambda-powertools-logger')

const defaultResults = process.env.defaultResults || 8
const tableName = process.env.restaurants_table

const getRestaurants = async (count) => {
  Log.debug('getting restaurants from DynamoDB...', {
    count,
    tableName
  })
  const req = {
    TableName: tableName,
    Limit: count
  }

  const resp = await dynamodb.scan(req).promise()
  return resp.Items
}

module.exports.handler = async (event, context) => {
  const restaurants = await getRestaurants(defaultResults)
  Log.debug('found restaurants', { count: restaurants.length })

  const response = {
    statusCode: 200,
    body: JSON.stringify(restaurants)
  }
  return response
}
```

4. Modify `functions/place-order.js` and replace `console.log` with use of the logger.

First, require the logger at the top of the file.

```javascript
const Log = require('@dazn/lambda-powertools-logger')
```

Replace the 2 instances of `console.log` with `Log.debug`.

On ln12:

```javascript
Log.debug('placing order...', { orderId, restaurantName })
```

On ln27:

```javascript
Log.debug(`published 'order_placed' event into EventBridge`, { busName })
```

4. Modify `functions/notify-restaurant.js` and replace `console.log` with use of the logger.

First, require the logger at the top of the file.

```javascript
const Log = require('@dazn/lambda-powertools-logger')
```

Replace the 2 instances of `console.log` with `Log.debug`.

On ln19:

```javascript
Log.debug('notified restaurant of order', { orderId, restaurantName })
```

On ln30:

```javascript
Log.debug(`published 'restaurant_notified' event to EventBridge`, { busName })
```

5. Run the integration tests

`STAGE=dev REGION=us-east-1 npm run test`

and see that the functions are now logging in JSON

```
  When we invoke the GET /restaurants endpoint
SSM params loaded
AWS credential loaded
{"message":"getting restaurants from DynamoDB...","count":8,"tableName":"restaurants-yancui","awsRegion":"us-east-1","environment":"dev","level":20,"sLevel":"DEBUG"}
{"message":"found restaurants","count":8,"awsRegion":"us-east-1","environment":"dev","level":20,"sLevel":"DEBUG"}
    ✓ Should return an array of 8 restaurants (357ms)

  When we invoke the GET / endpoint
{"message":"loading index.html...","awsRegion":"us-east-1","environment":"dev","level":30,"sLevel":"INFO"}
{"message":"loaded","awsRegion":"us-east-1","environment":"dev","level":30,"sLevel":"INFO"}
{"message":"getting restaurants...","url":"https://w3ms8yulw9.execute-api.us-east-1.amazonaws.com/dev/restaurants","awsRegion":"us-east-1","environment":"dev","level":20,"sLevel":"DEBUG"}
{"message":"got restaurants","count":8,"awsRegion":"us-east-1","environment":"dev","level":20,"sLevel":"DEBUG"}
    ✓ Should return the index page with 8 restaurants (402ms)

  When we invoke the notify-restaurant function
{"message":"notified restaurant of order","orderId":"e8e71bb8-f282-5c94-b3b0-afae04d9f61f","restaurantName":"Fangtasia","awsRegion":"us-east-1","environment":"dev","level":20,"sLevel":"DEBUG"}
{"message":"published 'restaurant_notified' event to EventBridge","busName":"order-events-bus-yancui","awsRegion":"us-east-1","environment":"dev","level":20,"sLevel":"DEBUG"}
    ✓ Should publish message to SNS
    ✓ Should publish event to EventBridge

  Given an authenticated user
[test-Jeffery-Hilton-&knCA7Th] - user is created
[test-Jeffery-Hilton-&knCA7Th] - initialised auth flow
[test-Jeffery-Hilton-&knCA7Th] - responded to auth challenge
    When we invoke the POST /orders endpoint
{"message":"placing order...","orderId":"585263cb-8e80-52ae-9b7c-86fd9c388d10","restaurantName":"Fangtasia","awsRegion":"us-east-1","environment":"dev","level":20,"sLevel":"DEBUG"}
{"message":"published 'order_placed' event into EventBridge","busName":"order-events-bus-yancui","awsRegion":"us-east-1","environment":"dev","level":20,"sLevel":"DEBUG"}
      ✓ Should return 200
      ✓ Should publish a message to EventBridge bus
[test-Jeffery-Hilton-&knCA7Th] - user deleted

  Given an authenticated user
[test-David-Carr-]c^Iwe&Z] - user is created
[test-David-Carr-]c^Iwe&Z] - initialised auth flow
[test-David-Carr-]c^Iwe&Z] - responded to auth challenge
    When we invoke the POST /restaurants/search endpoint with theme 'cartoon'
      ✓ Should return an array of 4 restaurants (251ms)
[test-David-Carr-]c^Iwe&Z] - user deleted


  7 passing (5s)
```

</p></details>

<details>
<summary><b>Disable debug logging in production</b></summary><p>

This logger allows you to control the default log level via the `LOG_LEVEL` environment variable. Let's configure the `LOG_LEVEL` environment such that we'll be logging at `INFO` level in production, but logging at `DEBUG` level everywhere else.

1. Open `serverless.yml`. Under the `custom` section, add `stage` and `logLevel` as below

```yml
stage: ${opt:stage, self:provider.stage}
logLevel:
  prod: INFO
  default: DEBUG
```

`custom.stage` uses the `${xxx, yyy}` syntax to provide fall backs. In this case, we're saying "if a `stage` variable is provided via the CLI, e.g. `sls deploy --stage staging`, then resolve to `staging`; otherwise, fallback to `provider.stage` in this file (hence the `self` reference"

2. Still in the `serverless.yml`, under `provider` section, add the following

```yml
environment:
  LOG_LEVEL: ${self:custom.logLevel.${self:custom.stage}, self:custom.logLevel.default}
```

After this change, the `provider` section should look like this:

```yml
provider:
  name: aws
  runtime: nodejs12.x
  stage: dev
  environment:
    LOG_LEVEL: ${self:custom.logLevel.${self:custom.stage}, self:custom.logLevel.default}
```

This applies the `LOG_LEVEL` environment variable (used to decide what level the logger should log at) to all the functions in the project (since it's specified under `provider`).

It references the `custom.logLevel` object (with the `self:` syntax), and also references the `custom.stage` value (remember, this can be overriden by CLI options). So when the deployment stage is `prod`, it resolves to `self:custom.logLevel.prod` and `LOG_LEVEL` would be set to `INFO`.

The second argument, `self:custom.logLevel.default` provides the fallback if the first path is not found. If the deployment stage is `dev`, it'll see that `self:custom.logLevel.dev` doesn't exist, and therefore use the fallback `self:custom.logLevel.default` and set `LOG_LEVEL` to `DEBUG` in that case.

This is a nice trick to specify a stage-specific override, but then fall back to some default value otherwise.

</p></details>
