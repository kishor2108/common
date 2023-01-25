# Local Development Setup
# 1.   `serverless.yml` Issue

## Description


```yml
environment:
      QUEUE_REFRESH_SQS_URL:
        Fn::GetAtt:
          - SqsQueueRefresh
          - QueueUrl
```
- In the `serverless.yml` file, we are using the AWS intrinsic function to assign the `QUEUE_REFRESH_SQS_URL` environment variable. However, please note that this variable will not be resolved in a local setup (as per the official documentation).
![We](https://raw.githubusercontent.com/kishor2108/common/master/unresolved_attr.png)
- And it's returning an error while running below command
```sh 
yarn run sls invoke local -f graphQL -p fixtures/v3/healthCheck.json -s local
```
# Error
```error
Could not resolve "QUEUE_REFRESH_SQS_URL" environment variable:
Unsupported environment variable format: { 'Fn::GetAtt': [ 'SqsQueueRefresh', 'QueueUrl' ] }
```

## Solution 
To Resolve this I added custom parameter like this
```yml
custom:
 {...}
 sqsEndPoint:
    Fn::GetAtt:
      - SqsQueueRefresh
      - QueueUrl
```

and assigned `QUEUE_REFRESH_SQS_URL` as below 
```yml
 environment:
      QUEUE_REFRESH_SQS_URL: ${param:sqsEndPoint, self:custom.sqsEndPoint}
```
Here we can pass extra params from the command 
```sh
yarn run sls invoke local -f graphQL -p fixtures/v3/healthCheck.json -s local --param="sqsEndPoint=http://localhost:9324/000000000000/portal-api-local-queue-refresh.fifo"
```
If we don't pass this params `QUEUE_REFRESH_SQS_URL`  will be assign from `custom.sqsEndPoint`

# 1.   `Webpack` Issue

```$
Webpack compilation failed:
in ../../node_modules/pg/lib/native/client.js 4:13-33
  Module not found: Error: Can't resolve 'pg-native' in '/home/bacancy/Documents/backend-monorepo/node_modules/pg/lib/native'
  .
  .
  ...
  Module not found: Error: Can't resolve 'pg-hstore' in '/home/bacancy/Documents/backend-monorepo/node_modules/sequelize/lib/dialects/postgres'
  .
  .
  ...
```
## Solution 

- I found that `pg-native` is `peerDependency` of `pg` package and also it's optional 
```json
 "peerDependencies": {
    "pg-native": ">=3.0.1"
  },
  "peerDependenciesMeta": {
    "pg-native": {
      "optional": true
    }
  },
```
- By Ignoring this two plugins it started working
```javascript
// webpack.config.js
// eslint-disable-next-line import/no-extraneous-dependencies
const webpack = require('webpack');
.
.
.
module.exports = {
    ...
  plugins: [new webpack.IgnorePlugin({ resourceRegExp: /^pg-native$/ }), new webpack.IgnorePlugin({ resourceRegExp: /^pg-hstore$/ })],
  ...
  .
  .
  .
};
```

# 3. Graphql Issue
Got Graphql error while running below command

```sh 
yarn run sls invoke local -f graphQL -p fixtures/v3/healthCheck.json -s local
```
# Error
```error
 Duplicate "graphql" modules cannot be used at the same time since different
 versions may have different capabilities and behavior. The data from one
 version used in the function from another could produce confusing and...
```

# Solution 

As of now couldn't find any solution but returning common API response object it's working fine.
```javascript
export const graphQL = (event, context, callback) => {
  if (event.source === 'serverless-plugin-warmup') {
    // eslint-disable-next-line no-console
    console.log('WarmUp - Lambda is warm!');
    return { status: 200, body: 'Lambda is warm!' };
  }
  return {
    isBase64Encoded: false,
    statusCode: 200,
    body: 'Hello from Lambda!',
    headers: {
      'content-type': 'application/json'
    }
  };
  // return createV3GraphQLHandlers(event, context, callback, false);
};
```
### NOTE: "Please note that when running the command yarn workspace portal-api sls offline start -s local, there is no need to make any changes to the GraphQL handler. The issue only occurs when invoking the local function."

# 4. Setting an Env Variable from the script

Please note that some of the scripts in the package.json file may set an environment variable such as NODE_ENV in the command line, but this method may not be supported on Windows systems. An alternative approach should be considered for setting environment variables on Windows.
```json
{
   
    "db:create:local": "yarn run build:sequelize && NODE_ENV=local sequelize-cli db:create",
  
}
```
