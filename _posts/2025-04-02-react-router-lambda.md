---
layout: post
title: React Router + AWS Lambda
date: 2025-04-02 09:20:00
author: Jon Winsley
summary: It's really easy to deploy a React Router app on AWS Lambda. This minimal demo configures the starter app and deploys it via AWS CDK.
---
Spinning up a [React Router](https://reactrouter.com/) app on AWS Lambda - with server side rendering and everything - is pretty easy. Here, we'll be using [the AWS CDK](https://aws.amazon.com/cdk/) to define and create the resources in AWS.

Here's [the working repo.](https://github.com/glitchassassin/react-router-lambda-demo)

## Set up React Router

For this demo, I'm just using `create-react-router@latest` to set up a new project:

```bash
npx create-react-router@latest ./react-router-lambda
```

For more complex projects, [the Epic Stack](https://github.com/epicweb-dev/epic-stack) is a great template to start from.

You can follow this article through the commit history in the `react-router-lambda` template repository. Here's [the first commit](https://github.com/glitchassassin/react-router-lambda-demo/commit/2b59fb392b75b5c1cb431f63e11fa1df583a3eb2).
### Add Lambda Handler

React Router itself isn't a server, but provides adapters for different servers. Install the `@react-router/architect` package:

```bash
npm install @react-router/architect
```

Then, the handler itself is a simple wrapper of the build:

```typescript
// server/lambda.ts

import { createRequestHandler } from "@react-router/architect";
// @ts-ignore (no types declared for build)
import * as build from "../build/server";

export const handler = createRequestHandler({
  build,
  mode: process.env.NODE_ENV,
});
```

This is all the code we're going to set up; check out [the commit here](https://github.com/glitchassassin/react-router-lambda-demo/commit/6bfe0eaf91291017a77f52b4eebdf868d64e4a67).

Now, let's dig into the CDK to make this run in AWS.
## Set up CDK

AWS CDK is a toolkit for defining infrastructure as code, which is great for consistent and reproducible deployments. We'll set up a few pieces: the server handler will run on Lambda; static client-side files (including CSS and Javascript) will be served from an S3 bucket; and a CloudFront distribution will split traffic between the two.

For the purposes of this demo, I'm going to get the initial files from `cdk init` in an empty directory and then integrate the generated files into the React Router app. In a production app with multiple services, you might want to maintain the infrastructure separately from the app.

I'm going to simplify things a bit and put the `bin/cdk.ts` and `lib/cdk-stack.ts` in `./cdk/`. I'm keeping the `cdk.json` file and adding the few necessary dependencies to `package.json`.

I did make one change in the `cdk.json` file, switching from `ts-node` to `tsx` for compatibility with the React Router Typescript setup:

```json
{
  "app": "npx tsx cdk/cdk.ts",
  // ...
```

Check out [the commit here](https://github.com/glitchassassin/react-router-lambda-demo/commit/914fe5f3bd4f641d899d3a20870a7178c9e69cd4) with the initial CDK setup.
### Lambda with Function URL

Time to set up the `cdk-stack`!

AWS provides a [`NodejsFunction`](https://docs.aws.amazon.com/cdk/api/v2/docs/aws-cdk-lib.aws_lambda_nodejs.NodejsFunction.html) construct that handles some of the packaging for us to build and deploy the server handler to Lambda. We'll use a function URL to invoke the Lambda rather than something like API Gateway.

```typescript
// Set absolute path to the lambda.ts handler
// __dirname is not available in ESM, but we can set it ourselves
const __dirname = path.dirname(fileURLToPath(import.meta.url));
const entry = path.join(__dirname, '../server/lambda.ts');

// Create Lambda function for React Router
const lambdaFunction = new nodejs.NodejsFunction(this, 'ReactRouterHandler', {
  runtime: lambda.Runtime.NODEJS_22_X,
  handler: 'handler',
  entry,
  bundling: {
    externalModules: [
      '@aws-sdk/*',
      'aws-sdk', // Not actually needed (or provided) - see below
    ],
    minify: true,
    sourceMap: true,
    target: 'es2022',
  },
  environment: {
    NODE_ENV: 'production',
  },
});

// Create Function URL for the Lambda
const functionUrl = lambdaFunction.addFunctionUrl({
  authType: lambda.FunctionUrlAuthType.NONE,
});
```

The old `aws-sdk` v2 dependency is included in the externalModules as a workaround - the `@react-router/architect` package has some out-of-date dependencies, but we don't actually need them for this.

If you deploy at this point, you can invoke the Lambda via its function URL! However, we're still missing the static resources. Let's publish those to an s3 bucket.

![The React Router demo page with no CSS or images](/assets/Pasted image 20250402074126.png)

### Static Bucket

Because the resources in the bucket will be served via CloudFront, the bucket itself does not need to be publicly accessible:

```typescript
// Create S3 bucket for static assets
const staticBucket = new s3.Bucket(this, 'StaticBucket', {
  enforceSSL: true,
  publicReadAccess: false,
  blockPublicAccess: s3.BlockPublicAccess.BLOCK_ALL,
  removalPolicy: cdk.RemovalPolicy.DESTROY,
  autoDeleteObjects: true,
});
```
### CloudFront Distribution

Then we'll set up the CloudFront distribution. Note that we are disabling caching on the default behavior (the lambda) and using optimized caching on the static bucket. We're also using a BucketDeployment to publish the static assets to the bucket: this will automatically reset the CloudFront distribution's cache when resources are published.

```typescript
// Create CloudFront distribution
const distribution = new cloudfront.Distribution(this, 'Distribution', {
  defaultBehavior: {
    origin: new origins.FunctionUrlOrigin(functionUrl),
    viewerProtocolPolicy: cloudfront.ViewerProtocolPolicy.REDIRECT_TO_HTTPS,
    allowedMethods: cloudfront.AllowedMethods.ALLOW_ALL,
    cachedMethods: cloudfront.CachedMethods.CACHE_GET_HEAD_OPTIONS,
    originRequestPolicy: cloudfront.OriginRequestPolicy.ALL_VIEWER_EXCEPT_HOST_HEADER,
    cachePolicy: cloudfront.CachePolicy.CACHING_DISABLED,
  },
  additionalBehaviors: {
    '/assets/*': {
      origin: origins.S3BucketOrigin.withOriginAccessControl(staticBucket, {
        originAccessLevels: [cloudfront.AccessLevel.READ, cloudfront.AccessLevel.LIST],
      }),
      viewerProtocolPolicy: cloudfront.ViewerProtocolPolicy.REDIRECT_TO_HTTPS,
      allowedMethods: cloudfront.AllowedMethods.ALLOW_GET_HEAD_OPTIONS,
      cachedMethods: cloudfront.CachedMethods.CACHE_GET_HEAD_OPTIONS,
      cachePolicy: cloudfront.CachePolicy.CACHING_OPTIMIZED,
    },
  },
});

// Deploy static assets to S3
new s3deploy.BucketDeployment(this, 'DeployStaticAssets', {
  sources: [s3deploy.Source.asset(path.join(__dirname, '../build/client/assets'))],
  destinationBucket: staticBucket,
  destinationKeyPrefix: 'assets',
  distribution,
  distributionPaths: ['/assets/*'],
});
```

Here's [the final commit](https://github.com/glitchassassin/react-router-lambda-demo/commit/2c298801c4c4eb46fe2e7a2c821f0c2a8b228fc3) with the CDK setup.
## Wrap up

And that's it! If you deploy these changes and visit the Cloudfront domain, you'll see the React Router splash page:

![The React Router demo page fully functional](/assets/Pasted image 20250402081738.png)

Obviously, there are more things you'd want to set up in a production stack - a custom domain, a WAF, perhaps a database, etc. But this minimal stack shows how easy it can be to deploy React Router to Lambda.