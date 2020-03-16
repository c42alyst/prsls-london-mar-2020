# Module 8: Writing integration tests

## Add integration tests

**Goal:** Write integration tests

<details>
<summary><b>Prepare tests</b></summary><p>

1. Add a `tests` folder to the project root

2. Add a `test_cases` folder under `tests`

3. Add a `steps` folder under `tests`

4. Install `chai` as a dev dependency

`npm install --save-dev chai`

5. Install `mocha` as a dev dependency

`npm install --save-dev mocha`

6. Install `cheerio` as a dev dependency

`npm install --save-dev cheerio`

7. Install `awscred` as a dependency

`npm install --save awscred`

8. Install `lodash` as a dependency

`npm install --save lodash`

9. Install `cross-env` as a dev dependency

`npm install --save-dev cross-env`

</p></details>

<details>
<summary><b>Add test case for get-index</b></summary><p>

1. Add `get_index.tests.js` file under `test_cases`

2. Modify `test_cases/get_index.tests.js` to the following

```javascript
const { expect } = require('chai')
const cheerio = require('cheerio')

describe(`When we invoke the GET / endpoint`, () => {
  it(`Should return the index page with 8 restaurants`, async () => {
    const res = await when.we_invoke_get_index()

    expect(res.statusCode).to.equal(200)
    expect(res.headers['Content-Type']).to.equal('text/html; charset=UTF-8')
    expect(res.body).to.not.be.null

    const $ = cheerio.load(res.body)
    const restaurants = $('.restaurant', '#restaurantsUl')
    expect(restaurants.length).to.equal(8)
  })
})
```

3. Add `when.js` file under `steps`

4. Modify `steps/when.js` to the following

```javascript
const APP_ROOT = '../../'
const _ = require('lodash')

const viaHandler = async (event, functionName) => {
  const handler = require(`${APP_ROOT}/functions/${functionName}`).handler

  const context = {}
  const response = await handler(event, context)
  const contentType = _.get(response, 'headers.Content-Type', 'application/json');
  if (response.body && contentType === 'application/json') {
    response.body = JSON.parse(response.body);
  }
  return response
}

const we_invoke_get_index = () => viaHandler({}, 'get-index')

module.exports = {
  we_invoke_get_index
}
```

5. Modify `test_cases/get-index.tests.js` to require the `when` module

```javascript
const { expect } = require('chai')
const cheerio = require('cheerio')
const when = require('../steps/when')

describe(`When we invoke the GET / endpoint`, () => {
```

6. Modify the `package.json` and add a `test` script

```json
"scripts": {
  "sls": "serverless",
  "test": "mocha tests/test_cases --reporter spec"
},
```

7. Run the integration test

`npm run test`

and see that the test fails with the error 

```
When we invoke the GET / endpoint
loading index.html...
loaded
    1) Should return the index page with 8 restaurants


  0 passing (42ms)
  1 failing

  1) When we invoke the GET / endpoint
       Should return the index page with 8 restaurants:
     TypeError [ERR_INVALID_ARG_TYPE]: The "url" argument must be of type string. Received type undefined
      at Url.parse (url.js:154:11)
      at Object.urlParse [as parse] (url.js:148:13)
      at getRestaurants (functions/get-index.js:27:19)
      at module.exports.handler (functions/get-index.js:43:29)
      at viaHandler (tests/steps/when.js:8:26)
      at Object.we_invoke_get_index (tests/steps/when.js:16:35)
      at Context.it (tests/test_cases/get_index.tests.js:7:28)
```

This is because the `get-index` function needs a number of environment variables, including the URL to the `get-restaurants` endpoint. We haven't set these up in our tests.

So what we can do, is to encapsulate all the initialization logic for our tests into its own module.

8. Add `init.js` under `steps` folder

9. Modify `init.js` to the following

```javascript
const { promisify } = require('util')
const awscred = require('awscred')

let initialized = false

const init = async () => {
  if (initialized) {
    return
  }

  process.env.restaurants_api      = "https://xxx.execute-api.us-east-1.amazonaws.com/dev/restaurants"
  process.env.restaurants_table    = "restaurants-yancui"
  process.env.AWS_REGION           = "us-east-1"
  process.env.cognito_user_pool_id = "test_cognito_user_pool_id"
  process.env.cognito_client_id    = "test_cognito_client_id"
  
  const { credentials } = await promisify(awscred.load)()
  
  process.env.AWS_ACCESS_KEY_ID     = credentials.accessKeyId
  process.env.AWS_SECRET_ACCESS_KEY = credentials.secretAccessKey

  console.log('AWS credential loaded')

  initialized = true
}

module.exports = {
  init
}
```

**IMPORTANT** replace the `restaurants_api` url on ln11 with the actual restaurnats URL, which you can find from the last time you deploy the project.

**IMPORTANT** replace the `cognito_user_pool_id` and `cognito_client_id` with the user pool ID (not the `arn`) from before, and use the client id for the `web` client. You can find both in the `serverless.yml`.

10. Modify `test_cases/get-index.tests.js` to require the `init` module at the top of the file

```javascript
const { expect } = require('chai')
const cheerio = require('cheerio')
const when = require('../steps/when')
const { init } = require('../steps/init')

describe(`When we invoke the GET / endpoint`, () => {
```

11. Modify `test_cases/get-index.tests.js` to add a `before` step in the test case `When we invoke the GET / endpoint`

```javascript
describe(`When we invoke the GET / endpoint`, () => {
  before(async () => await init())

  it(`Should return the index page with 8 restaurants`, async () => {
```

12. Run the integration test

`npm run test`

and see that the test passes

```
  When we invoke the GET / endpoint
AWS credential loaded
loading index.html...
loaded
    ✓ Should return the index page with 8 restaurants (449ms)


  1 passing (467ms)
```

</p></details>

If you find that some tests fails sporadically, they might just need longer to run. Try raising the test timeout with `--timeout 5000`, the default timeout is 2s. You can do this by updating the `test` script in the `package.json`, i.e.

```
"scripts": {
  "sls": "serverless",
  "test": "mocha tests/test_cases --reporter spec --timeout 5000"
},
```

## Exercises

<details>
<summary><b>Add test case for get-restaurants</b></summary><p>

1. Add `get-restaurants.tests.js` under `test_cases`

2. Modify `test_cases/get-restaurants.tests.js` to the following

```javascript
const { expect } = require('chai')
const { init } = require('../steps/init')
const when = require('../steps/when')

describe(`When we invoke the GET /restaurants endpoint`, () => {
  before(async () => await init())

  it(`Should return an array of 8 restaurants`, async () => {
    let res = await when.we_invoke_get_restaurants()

    expect(res.statusCode).to.equal(200)
    expect(res.body).to.have.lengthOf(8)

    for (let restaurant of res.body) {
      expect(restaurant).to.have.property('name')
      expect(restaurant).to.have.property('image')
    }
  })
})
```

3. Modify `when.js` to add a `we_invoke_get_restaurants` function

```javascript
const we_invoke_get_restaurants = () => viaHandler({}, 'get-restaurants')

module.exports = {
  we_invoke_get_index,
  we_invoke_get_restaurants
}
```

4. Run the integration test

`npm run test`

and see that the tests pass

```
  When we invoke the GET / endpoint
AWS credential loaded
loading index.html...
loaded
    ✓ Should return the index page with 8 restaurants (371ms)

  When we invoke the GET /restaurants endpoint
    ✓ Should return an array of 8 restaurants (451ms)


  2 passing (839ms)
```

</p></details>

<details>
<summary><b>Add test case for search-restaurants</b></summary><p>

1. Add `search-restaurants.tests.js` under `test_cases`

2. Modify `test_cases/search-restaurants.tests.js` to the following

```javascript
const { expect } = require('chai')
const { init } = require('../steps/init')
const when = require('../steps/when')

describe(`When we invoke the POST /restaurants/search endpoint with theme 'cartoon'`, () => {
  before(async () => await init())

  it(`Should return an array of 4 restaurants`, async () => {
    let res = await when.we_invoke_search_restaurants('cartoon')

    expect(res.statusCode).to.equal(200)
    expect(res.body).to.have.lengthOf(4)

    for (let restaurant of res.body) {
      expect(restaurant).to.have.property('name')
      expect(restaurant).to.have.property('image')
    }
  })
})
```

3. Modify `when.js` to add a `we_invoke_search_restaurants` function

```javascript
const we_invoke_search_restaurants = theme => {
  let event = { 
    body: JSON.stringify({ theme })
  }
  return viaHandler(event, 'search-restaurants')
}

module.exports = {
  we_invoke_get_index,
  we_invoke_get_restaurants,
  we_invoke_search_restaurants
}
```

4. Run the integration test

`npm run test`

and see that the tests pass

```
  When we invoke the GET / endpoint
AWS credential loaded
loading index.html...
loaded
    ✓ Should return the index page with 8 restaurants (435ms)

  When we invoke the GET /restaurants endpoint
    ✓ Should return an array of 8 restaurants (440ms)

  When we invoke the POST /restaurants/search endpoint with theme 'cartoon'
    ✓ Should return an array of 4 restaurants (249ms)


  3 passing (1s)
```

</p></details>

<details>
<summary><b>How would you improve the get-index test so it doesn't rely on there being data in the database already?</b></summary><p>

</p></details>

<details>
<summary><b>What other test cases would you add?</b></summary><p>

</p></details>