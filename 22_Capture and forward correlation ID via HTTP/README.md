# Module 22: Capture and forward correlation ID

## Capture and forward correlation IDs

Since we're using the `@dazn/lambda-powertools-pattern-basic` and `@dazn/lambda-powertools-logger`, we are already capturing incoming correlation IDs and including them in the logs.

However, we need to make sure the `get-index` function forwards the correlation IDs along, while still using the X-Ray instrumented `https` module.

The `dazn-lambda-powertools` project also has Kinesis and SNS clients that will automatically forward captured correlation IDs. We can use them as stand-in replacements for the Kinesis and SNS clients from the AWS SDK.

<details>
<summary><b>Forward correlation IDs from get-index</b></summary><p>

You can access the auto-captured correlation IDs using the `@dazn/lambda-powertools-correlation-ids` package. From here, we can include them as HTTP headers.

1. At the project root, run `npm install --save @dazn/lambda-powertools-correlation-ids`.

2. Open `functions/get-index.js` and require the `@dazn/lambda-powertools-correlation-ids` module (at the top of the file).

```javascript
const CorrelationIds = require('@dazn/lambda-powertools-correlation-ids')
```

3. Staying in `functions/get-index.js`, replace the `getRestaurants` function with the following

```javascript
const getRestaurants = async () => {
  Log.debug('getting restaurants...', { url: restaurantsApiRoot })

  const url = URL.parse(restaurantsApiRoot)
  const opts = {
    host: url.hostname,
    path: url.pathname
  }

  aws4.sign(opts)

  const httpReq = http.get(restaurantsApiRoot, {
    headers: Object.assign({}, opts.headers, CorrelationIds.get())
  })
  const restaurants = (await httpReq).data
  Log.debug('got restaurants', { count: restaurants.length })

  return restaurants
}
```

These changes are enough to ensure correlation IDs are included in the HTTP headers in the request to the `GET /restaurants` endpoint.

4. Open `tests/steps/when.js` and modify the `viaHandler` method so that `context` is initialized with a `awsRequestId`, e.g.

`const context = { awsRequestId: 'test' }`

This will be used to initialize the correlation ID with and you'll see it in the logs when you run the test

`STAGE=dev REGION=us-east-1 npm run test`

```
  When we invoke the GET / endpoint
SSM params loaded
AWS credential loaded
invoking via handler function get-index
{"message":"loading index.html...","awsRegion":"us-east-1","environment":"dev","awsRequestId":"test","x-correlation-id":"test","debug-log-enabled":"false","call-chain-length":1,"level":30,"sLevel":"INFO"}
{"message":"loaded","awsRegion":"us-east-1","environment":"dev","awsRequestId":"test","x-correlation-id":"test","debug-log-enabled":"false","call-chain-length":1,"level":30,"sLevel":"INFO"}
    ✓ Should return the index page with 8 restaurants (1664ms)

  ...
```

5. Redeploy the project.

`npm run sls -- deploy -s dev -r us-east-1`

6. Once the deployment is done, load the page. And then open the X-Ray console to make sure that the X-Ray tracing is still working.

7. Open the CloudWatch console to check the logs for both `get-index` and `get-restaurants`. You should see that the same correlation ID is included in both logs.

</p></details>

<details>
<summary><b>Forward correlation IDs through EventBridge events</b></summary><p>

The `dazn-lambda-powertools` suite has a number of like-for-like replacement packages for AWS SDK clients that can automatically forward correlation IDs that has been captured by the wrapper.

So, to forward correlation IDs from the `place-order` function onto `notify-restaurant` through EventBridge, we need to use the `@dazn/lambda-powertools-eventbridge-client` NPM package.

1. Install the `@dazn/lambda-powertools-eventbridge-client`

`npm install @dazn/lambda-powertools-eventbridge-client`

2. In `place-order.js` file, replace

`const eventBridge = new AWS.EventBridge()`

with this:

```javascript
const eventBridge = process.env.LAMBDA_RUNTIME_DIR
  ? AWSXRay.captureAWSClient(require('@dazn/lambda-powertools-eventbridge-client'))
  : require('@dazn/lambda-powertools-eventbridge-client')
```

3. Redeploy

`npm run sls -- deploy -s dev -r us-east-1`

and place a few orders, and then check the logs for `place-order` and `notify-restaurant`. You should see on a few occassions (remember, debug logs are sampled at 10%) that debug logs are enabled on the whole transaction and that both functions have the same `x-correlation-id`.

e.g. in the `place-order` function:
![](/images/mod22-003.png)

the same `x-correlation-id` is recorded in `notify-restaurant` function, and note the decision to enable debug logging is passed along
![](/images/mod22-004.png)

4. Also, check out [**this repo**](https://github.com/theburningmonk/lambda-distributed-tracing-demo/tree/master/lambda-powertools) to see a more comprehensive demo of how you can auto-extract and forward correlation IDs through a variety of different event sources with the dazn-lambda-powertools.

</p></details>
