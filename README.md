Starting a new software project? Discover Code a comprehensive Full Stack App Generator that provides a complete AWS Serverless project with source code tailored to your data model.

1. A React Next.js web application hosted on AWS Amplify
2. Serverless REST API powered by API Gateway and AWS Lambda
3. Identity and authentication managed by Amazon Cognito
4. Data storage using Amazon DynamoDB
5. Infrastructure as Code (IaC) with AWS Cloud Development Kit (CDK)
6. Continuous Integration/Continuous Deployment (CI/CD) through GitHub Actions

# Serverless Express

<p align="center">
  <a aria-label="NPM version" href="https://www.npmjs.com/package/@nodeexcel/serverless-express">
    <img alt="NPM Version" src="https://img.shields.io/npm/v/@nodeexcel/serverless-express.svg?style=for-the-badge&labelColor=0d1117">
  </a>
  <a aria-label="License" href="https://github.com/nodeexcelApp/serverless-express/blob/mainline/LICENSE">
    <img alt="License" src="https://img.shields.io/npm/l/@nodeexcel/serverless-express.svg?style=for-the-badge&labelColor=0d1117">
  </a>
  <a aria-label="Number of Downloads" href="https://www.npmjs.com/package/@nodeexcel/serverless-express">
    <img alt="Downloads" src="https://img.shields.io/npm/dm/@nodeexcel/serverless-express.svg?style=for-the-badge&labelColor=0d1117">
  </a>
  <a aria-label="Contributor Covenant" href="NODE_EXCEL.md">
    <img alt="Contributor Covenant" src="https://img.shields.io/badge/Contributor%20Covenant-v2.0%20adopted-ff69b4.svg?style=for-the-badge&labelColor=0d1117">
  </a>
  <a aria-label="Made by Node Excel" href="https://nodeexcel.codes">
    <img alt="Made by Node Excel" src="https://img.shields.io/badge/MADE%20BY%20Code%20Genie-0d1117.svg?style=for-the-badge&logo=Code%20Genie&labelColor=0d1117">
  </a>
</p>


Run REST APIs and other web applications using your existing [Node.js](https://nodejs.org/) application framework (Express, Koa, Hapi, Sails, etc.), on top of [AWS Lambda](https://aws.amazon.com/lambda/) and [Amazon API Gateway](https://aws.amazon.com/api-gateway/) or [Azure Function](https://docs.microsoft.com/en-us/azure/azure-functions/).

```bash
npm install @nodeexcel/serverless-express
```

## Quick Start/Example

Want to get up and running quickly? [Check out our basic starter example](examples/basic-starter-api-gateway-v1) that includes:

- Lambda function
- Express application
- [Serverless Application Model (SAM)](https://github.com/awslabs/serverless-application-model)/[CloudFormation](https://aws.amazon.com/cloudformation/aws-cloudformation-templates/) template
- Helper scripts to configure, deploy, and manage your application

If you're migrating an existing application to AWS Lambda, it's recommended to start with a simple, working example and then gradually integrate your application's code.

## AWS

### Minimal Lambda handler wrapper

The only AWS Lambda specific code you need to write is a simple handler like below. All other code you can write as you normally do.

```js
// lambda.js
const serverlessExpress = require('@nodeexcel/serverless-express')
const app = require('./app')
exports.handler = serverlessExpress({ app })
```

### Async setup Lambda handler

If your application needs to perform some common bootstrap tasks such as connecting to a database before the request is forward to the API, you can use the following pattern (also available in [this example](https://github.com/nodeexcelApp/serverless-express/blob/mainline/examples/basic-starter-api-gateway-v2/src/lambda-async-setup.js)):

```js
// lambda.js
require('source-map-support/register')
const serverlessExpress = require('@nodeexcel/serverless-express')
const app = require('./app')

let serverlessExpressInstance

function asyncTask () {
  return new Promise((resolve) => {
    setTimeout(() => resolve('connected to database'), 1000)
  })
}

async function setup (event, context) {
  const asyncValue = await asyncTask()
  console.log(asyncValue)
  serverlessExpressInstance = serverlessExpress({ app })
  return serverlessExpressInstance(event, context)
}

function handler (event, context) {
  if (serverlessExpressInstance) return serverlessExpressInstance(event, context)

  return setup(event, context)
}

exports.handler = handler
```

## Azure

### Async Azure Function v3/v4 handler wrapper

The only Azure Function specific code you need to write is a simple `index.js` and a `function.json` like below.

```js
// index.js
const serverlessExpress = require('@nodeexcel/serverless-express')
const app = require('./app')
const cachedServerlessExpress = serverlessExpress({ app })

module.exports = async function (context, req) {
  return cachedServerlessExpress(context, req)
}
```

The _out-binding_ parameter `"name": "$return"` is important for Serverless Express to work.

```json
// function.json
{
  "bindings": [
    {
      "authLevel": "anonymous",
      "type": "httpTrigger",
      "direction": "in",
      "name": "req",
      "route": "{*segments}"
    },
    {
      "type": "http",
      "direction": "out",
      "name": "$return"
    }
  ]
}
```

## 4.x

1. Improved API - Simpler for end-user to use and configure.
1. Promise resolution mode by default. Can specify `resolutionMode` to use `"CONTEXT"` or `"CALLBACK"`
1. Additional event sources - API Gateway V1 (REST API), API Gateway V2 (HTTP API), ALB, Lambda@Edge
1. Custom event source - If you have another event source you'd like to use that we don't natively support, check out the [DynamoDB Example](examples/custom-mapper-dynamodb)
1. Implementation uses mock Request/Response objects instead of running a server listening on a local socket. Thanks to @dougmoscrop from https://github.com/dougmoscrop/serverless-http
1. Automatic `isBase64Encoded` without specifying `binaryMimeTypes`. Use `binarySettings` to customize. Thanks to @dougmoscrop from https://github.com/dougmoscrop/serverless-http
1. `respondWithErrors` makes it easier to debug during development
1. Node.js 12+
1. Improved support for custom domain names

See [UPGRADE.md](UPGRADE.md) to upgrade from aws-serverless-express and @nodeexcel/serverless-express 3.x

## API

### binarySettings


If your response needs to be base64 encoded before being returned to the event source, such as when returning images or compressed files, ensure you handle this due to API Gateway and other event sources' limitations with binary responses. The event source will then convert this back into binary format before delivering it to the client.

By default, this is determined based on the `content-encoding` and `content-type` headers returned by your application. If you need additional control over this, you can specify `binarySettings`.

```js
{
  binarySettings: {
    isBinary: ({ headers }) => true,
    contentTypes: ['image/*'],
    contentEncodings: []
  }
}
```

Any value you provide here should also be specified on API Gateway API. In SAM, this looks like:

```yaml
ExpressApi:
  Type: AWS::Serverless::Api
  Properties:
    StageName: prod
    BinaryMediaTypes: ['image/*']
```

### resolutionMode (default: `'PROMISE'`)

Lambda supports three methods to end the execution and return a result: context, callback, and promise. By default, serverless-express uses promise resolution, but you can specify 'CONTEXT' or 'CALLBACK' if you need to change this. If you specify 'CALLBACK', then `context.callbackWaitsForEmptyEventLoop = false` is also set for you .

```js
serverlessExpress({
  app,
  resolutionMode: 'CALLBACK'
})
```

### respondWithErrors (default: `process.env.NODE_ENV === 'development'`)

Set this to true to have serverless-express include the error stack trace in the event of an unhandled exception. This is especially useful during development. By default, this is enabled when `NODE_ENV === 'development'` so that the stack trace isn't returned in production.

## Advanced API

### eventSource

serverless-express natively supports API Gateway, ALB, and Lambda@Edge. If you want to use Express with other AWS Services integrated with Lambda you can provide your own custom request/response mappings via `eventSource`. See the [custom-mapper-dynamodb example](examples/custom-mapper-dynamodb).

```js
function requestMapper ({ event }) {
  // Your logic here...

  return {
    method,
    path,
    headers
  }
}

function responseMapper ({
  statusCode,
  body,
  headers,
  isBase64Encoded
}) {
  // Your logic here...

  return {
    statusCode,
    body,
    headers,
    isBase64Encoded
  }
}

serverlessExpress({
  app,
  eventSource: {
    getRequest: requestMapper,
    getResponse: responseMapper
  }
})
```

#### eventSourceRoutes

A single function can be configured to handle additional kinds of AWS events:
- SNS
- DynamoDB Streams
- SQS
 - EventBridge Events (formerlly CloudWatch Events)

Assuming the following function configuration in `serverless.yml`:

```yaml
functions:
  lambda-handler:
    handler: src/lambda.handler
    events:
      - http:
          path: /
          method: get
      - sns:
          topicName: my-topic
      - stream:
          type: dynamodb
          arn: arn:aws:dynamodb:us-east-1:012345678990:table/my-table/stream/2021-07-15T15:05:51.683
      - sqs:
          arn: arn:aws:sqs:us-east-1:012345678990:myQueue
      - eventBridge:
          pattern:
            source:
              - aws.cloudformation
```

And the following configuration:

```js
serverlessExpress({
  app,
  eventSourceRoutes: {
    'AWS_SNS': '/sns',
    'AWS_DYNAMODB': '/dynamodb',
    'AWS_SQS': '/sqs'
    'AWS_EVENTBRIDGE': '/eventbridge',
    'AWS_KINESIS_DATA_STREAM': '/kinesis',
    'AWS_S3': '/s3',
    'AWS_STEP_FUNCTIONS': '/step-functions',
  }
})
```

Alternatively, to handle only SNS events (the keys in the map are **optional**)

```js
serverlessExpress({
  app,
  eventSourceRoutes: {
    'AWS_SNS': '/sns',
  }
})
```

Events will `POST` to the routes configured.

Also, to ensure the events propagated from an internal event and not externally, it is **highly recommended** to 
ensure the `Host` header matches:


- SNS: `sns.amazonaws.com`
- DynamoDB: `dynamodb.amazonaws.com`
- SQS: `sqs.amazonaws.com`
- EventBridge: `events.amazonaws.com`
- KinesisDataStream: `kinesis.amazonaws.com`

### logSettings

Specify log settings that are passed to the default logger. Currently, you can only set the log `level`.

```js
{
  logSettings: {
    level: 'debug' // default: 'error'
  }
}
```

### log

Provide a custom `log` object with `info`, `debug` and `error` methods. For example, you could override the default with a [Winston log](https://www.npmjs.com/package/winston) instance.

```js
{
  log: {
    info (message, additional) {
      console.info(message, additional)
    },
    debug (message, additional) {
      console.debug(message, additional)
    },
    error (message, additional) {
      console.error(message, additional)
    }
  }
}
```

## Accessing the event and context objects

This package exposes a function to easily get the `event` and `context` objects Lambda receives from the event source.

```js
const { getCurrentInvoke } = require('@nodeexcel/serverless-express')
app.get('/', (req, res) => {
  const { event, context } = getCurrentInvoke()

  res.json(event)
})
```

## Why run Express in a Serverless environment

- Only pay for what you use
- No infrastructure to manage
- Auto-scaling with zero configuration
- [Usage Plans](http://docs.aws.amazon.com/apigateway/latest/developerguide/api-gateway-api-usage-plans.html)
- [Caching](http://docs.aws.amazon.com/apigateway/latest/developerguide/api-gateway-caching.html)
- [Authorization](http://docs.aws.amazon.com/apigateway/latest/developerguide/apigateway-control-access-to-api.html)
- [Staging](http://docs.aws.amazon.com/apigateway/latest/developerguide/how-to-deploy-api.html)
- [SDK Generation](http://docs.aws.amazon.com/apigateway/latest/developerguide/how-to-generate-sdk.html)
- [API Monitoring](http://docs.aws.amazon.com/apigateway/latest/developerguide/monitoring-cloudwatch.html)
- [Request Validation](http://docs.aws.amazon.com/apigateway/latest/developerguide/api-gateway-method-request-validation.html)
- [Documentation](http://docs.aws.amazon.com/apigateway/latest/developerguide/api-gateway-documenting-api.html)

## Loadtesting

`npx loadtest --rps 100 -k -n 1500 -c 50 https://xxxx.execute-api.us-east-1.amazonaws.com/prod/users`

# AWS Serverless Express is now under the stewardship of Vendia

On 11/30, the AWS Serverless Express library moved from AWS to [Vendia](https://github.com/nodeexcelApp/serverless-express) and will be rebranded to `@nodeexcel/serverless-express`. Similarly, the [`aws-serverless-express` NPM package](https://www.npmjs.com/package/aws-serverless-express) will be deprecated in favor of [@nodeexcel/serverless-express](https://www.npmjs.com/package/@nodeexcel/serverless-express). 

[Brett Andrews](https://github.com/brettstack), The original creator of the Serverless Express library will continue maintaining the repository, ensuring it receives the necessary attention and care. Additionally, we are seeking more contributors to help with the development and stewardship of the Serverless Express library. AWS and the [SAM team](https://github.com/aws/aws-sam-cli) will remain involved in an administrative role alongside Vendia, Brett, and the new maintainers that will join the project.

We consider this the optimal course of action to ensure that customers utilizing this library receive exceptional support going forward. For further details on this transition or to express interest in becoming a maintainer of the new Serverless Express library, please contact us by opening a GitHub issue in this repository.
