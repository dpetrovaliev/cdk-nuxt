# AWS CDK Nuxt 3 Deployment Stack

<p>
    <a href="https://github.com/ferdinandfrank/cdk-nuxt/actions/workflows/publish.yml"><img alt="Build" src="https://img.shields.io/github/actions/workflow/status/ferdinandfrank/cdk-nuxt/publish.yml?logo=github" /></a>
    <a href="https://www.npmjs.com/package/cdk-nuxt"><img alt="Version" src="https://img.shields.io/npm/v/cdk-nuxt.svg" /></a>
    <a href="https://www.npmjs.com/package/cdk-nuxt"><img alt="License" src="https://img.shields.io/npm/l/cdk-nuxt.svg" /></a>
</p>

Easily deploy Nuxt 3 applications via CDK on AWS including the following features:

- Fast responses via [AWS Lambda](https://aws.amazon.com/lambda/)
- Publicly available by a custom domain (or subdomain) via [Route53](https://aws.amazon.com/route53/) and [API Gateway](https://aws.amazon.com/api-gateway/)
- Automatic redirects from HTTP to HTTPS via [CloudFront](https://aws.amazon.com/cloudfront/)
- Automatic upload of the build files for CSR and static assets to [S3](https://aws.amazon.com/s3/) with optimized caching rules
- Scheduled pings of the Nuxt app to keep the Lambda warm for fast responses via [EventBridge](https://aws.amazon.com/eventbridge/) rules
- Automatic cleanup of outdated static assets and build files

## Prerequisites

- This package currently relies on using [yarn](https://yarnpkg.com/) instead of npm for deployment. Therefore, make sure to have yarn available on the deployment system.
- You need an [AWS account](https://aws.amazon.com/premiumsupport/knowledge-center/create-and-activate-aws-account/?nc1=h_ls) to create and deploy the required resources for the Nuxt app on AWS.

## Installation

Install the package and its required dependencies:
```bash
yarn add cdk-nuxt --dev # The package itself
yarn add ts-node typescript --dev # To compile the CDK stacks via typescript
yarn add aws-cdk@2.88.0 --dev # CDK cli with this exact version for the deployment
```

## Setup

1. Set the Nitro preset on your Nuxt configuration file (`nuxt.config.js`) to `aws-lambda`:
    ```js
    export default defineNuxtConfig({
        ...
        nitro: {
            preset: 'aws-lambda'
        },
        ...      
    });
    ```
   See https://nitro.unjs.io/deploy/providers/aws for more details.
2. Remove `"type": "module"` from your `package.json` file, if it exists.
   This is required to make the CDK stack work. Click [here](https://github.com/ferdinandfrank/cdk-nuxt/issues/3) for details. 
3. [Create an AWS account](https://aws.amazon.com/premiumsupport/knowledge-center/create-and-activate-aws-account/?nc1=h_ls), if you don't have one yet. Then login into the AWS console and note the `Account ID`. You will need it in step 7.
4. [Create a hosted zone in Route53](https://docs.aws.amazon.com/Route53/latest/DeveloperGuide/AboutHZWorkingWith.html) for the desired domain, if you don't have one yet.<br/>This is required to create DNS records for the domain to make the Nuxt app publicly available on that domain.<br/>On the hosted zone details you should see the `Hosted zone ID` of the hosted zone. You will need it in step 7.
5. [Request a public **regional** certificate in the AWS Certificate Manager (ACM)](https://docs.aws.amazon.com/acm/latest/userguide/gs-acm-request-public.html) for the desired domain in your desired region, e.g., `eu-central-1`, and validate it, if you don't have one yet.<br/>This is required to make the Nuxt app accessible via the custom domain and to provide the custom domain to the Nuxt app via the 'Host' header for server side rendering use cases.<br/>Take note of the displayed `ARN` for the certificate. You will need it in step 7.
6. [Request a public **global** certificate in the AWS Certificate Manager (ACM)](https://docs.aws.amazon.com/acm/latest/userguide/gs-acm-request-public.html) for the desired domain in `us-east-1` (**global**) and validate it, if you don't have one yet.<br/>This is required to provide the Nuxt app via HTTPS on the public internet.<br/>Take note of the displayed `ARN` for the certificate. You will need it in step 7.<br/>**Important: The certificate must be issued in us-east-1 (global) regardless of the region used for the Nuxt app itself as it will be attached to the Cloudfront distribution which works globally.**
7. Run the following command to automatically create the required CDK stack entrypoint at `stack/index.ts`. This file defines the config how the Nuxt app will be deployed via CDK. You should adapt the file to the project's needs, especially the props `env.account` (setup step 3), `hostedZoneId` (setup step 4), `regionalTlsCertificateArn` (setup step 5) and `globalTlsCertificateArn` (setup step 6).

   ```bash
   node_modules/.bin/cdk-nuxt-init-server
   ```

   > :warning: It's recommended using a `.env` file or another secrets file to import the sensitive secrets into the `stack/index.ts` file.

## Configuration

The `NuxtServerAppStack` construct can be configured via the following props:

### project: string
A string identifier for the project the Nuxt app is part of.
A project might have multiple different services.

### service: string
A string identifier for the project's service the Nuxt app is created for.
This can be seen as the name of the Nuxt app.

### environment: string
A string to identify the environment of the Nuxt app. This enables us
to deploy multiple different environments of the same Nuxt app, e.g., production and development.

### domain: string
The domain (without the protocol) at which the Nuxt app shall be publicly available.
A DNS record will be automatically created in Route53 for the domain.
This also supports subdomains.
Examples: "example.com", "sub.example.com"

### hostedZoneId: string
The id of the hosted zone to create a DNS record for the specified domain.

### globalTlsCertificateArn: string
The ARN of the certificate to use on CloudFront for the Nuxt app to make it accessible via HTTPS.
The certificate must be issued for the specified domain in us-east-1 (global) regardless of the
region specified via 'env.region' as CloudFront only works globally.

### regionalTlsCertificateArn: string
The ARN of the certificate to use at the ApiGateway for the Nuxt app to make it accessible via the custom domain
and to provide the custom domain to the Nuxt app via the 'Host' header for server side rendering use cases.
The certificate must be issued in the same region as specified via 'env.region' as ApiGateway works regionally.

### rootDir?: string;
The path to the root directory of the Nuxt app (at which the `nuxt.config.ts` file is located).
Defaults to '.'.

### entrypoint?: string
The file name (without extension) of the Lambda entrypoint within the 'server' directory exporting a handler.
Defaults to "index".

### entrypointEnv?: string
A JSON serialized string of environment variables to pass to the Lambda function.

### memorySize?: number
The memory size to apply to the Nuxt app's Lambda.
Defaults to 1792MB (optimized for costs and performance for standard Nuxt apps).

### enableTracing?: boolean
Whether to enable AWS X-Ray for the Nuxt Lambda function.

### enableSitemap?: boolean
Whether to enable a global Sitemap bucket which is permanently accessible through multiple deployments.

### outdatedAssetsRetentionDays?: boolean
The number of days to retain static assets of outdated deployments in the S3 bucket.
Useful to allow users to still access old assets after a new deployment when they are still browsing on an old version.
Defaults to 30 days.

### allowHeaders?: string[]
An array of headers to pass to the Nuxt app on SSR requests.
The more headers are passed, the weaker the cache performance will be, as the cache key
is based on the headers.
No headers are passed by default.

### allowCookies?: string[]
An array of cookies to pass to the Nuxt app on SSR requests.
The more cookies are passed, the weaker the cache performance will be, as the cache key
is based on the cookies.
No cookies are passed by default.

### allowQueryParams?: string[]
An array of query param keys to pass to the Nuxt app on SSR requests.
The more query params are passed, the weaker the cache performance will be, as the cache key
is based on the query params.
Note that this config can not be combined with {@see denyQueryParams}.
If both are specified, the {@see denyQueryParams} will be ignored.
All query params are passed by default.

### denyQueryParams?: string[]
An array of query param keys to deny passing to the Nuxt app on SSR requests.
It might be useful to prevent specific external query params, e.g., fbclid, utm_campaign, ...,
to improve cache performance, as the cache key is based on the specified query params.
Note that this config can not be combined with {@see allowQueryParams}.
If both are specified, the {@see denyQueryParams} will be ignored.
All query params are passed by default.


## Deployment

After the installation and the setup you are already good to go to build the Nuxt app and to deploy it to AWS with this package
by following the steps below:

### 1. Bootstrap CDK
Deploying stacks with the AWS CDK requires dedicated Amazon S3 buckets and other containers to be available to AWS CloudFormation during deployment. 
Creating these is called bootstrapping and is **only required once** per account and region. 
To bootstrap, run the following command:

```bash
cdk bootstrap aws://ACCOUNT-NUMBER/REGION
```

See https://docs.aws.amazon.com/cdk/v2/guide/getting_started.html for details.

### 2. Build and Deploy

By running the following script, the packages of the Nuxt app will be installed and the Nuxt app will be built automatically.
Afterwards, the CDK stack will be deployed to AWS.

```bash
node_modules/.bin/cdk-nuxt-deploy-server
```

## Optional: Automatically deploy on every push (CD) via [GitHub Actions](https://github.com/features/actions)

Feel free to copy the following GitHub Actions YAML file content into a YAML file at `.github/workflows/deploy.yml` to automatically build and deploy the Nuxt app to AWS on every push to a specific branch.<br/>This only works if you're using GitHub for the project's VCS repository.

```yaml
name: Deploy

on:
  push:
    branches:
      - master # Feel free to use another branch name

jobs:
  build:
    runs-on: ubuntu-latest
    strategy:
      matrix:
        node-version: [ 16.x ]
    steps:
      - name: Checkout source code
        uses: actions/checkout@v2

      - name: Use Node.js ${{ matrix.node-version }}
        uses: actions/setup-node@v1
        with:
          node-version: ${{ matrix.node-version }}

      # Init cache to speed up yarn install
      - name: Get yarn cache directory path
        id: yarn-cache-dir-path
        run: echo "::set-output name=dir::$(yarn cache dir)"

      - uses: actions/cache@v2
        with:
          path: ${{ steps.yarn-cache-dir-path.outputs.dir }}
          key: ${{ runner.os }}-node-${{ hashFiles('**/yarn.lock') }}
          restore-keys: |
            ${{ runner.os }}-node-

      - name: Install dependencies
        run: yarn install --frozen-lockfile

      - name: Build and deploy to AWS
        run: node_modules/.bin/cdk-nuxt-deploy-server
        env:
           # Create an IAM user on AWS for the deployment and create the appropriate secrets in the GitHub repository secrets
          AWS_ACCESS_KEY_ID: ${{ secrets.AWS_ACCESS_KEY_ID }}
          AWS_SECRET_ACCESS_KEY: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
          AWS_DEFAULT_REGION: ${{ secrets.AWS_DEFAULT_REGION }}
```

## Advanced: Used AWS Resources

### NuxtServerAppStack

This stack is responsible for deploying dynamic Nuxt 3 apps to AWS.
The following AWS resources will be created by this stack:

- [Lambda](https://aws.amazon.com/lambda/): 
  - A Lambda function to render the Nuxt app including a separated Lambda layer to provide the `node_modules` of the Nuxt app required for server-side rendering.
  - A Lambda function that deletes the outdated static assets of the Nuxt app from S3.
- [S3](https://aws.amazon.com/s3/): A bucket to store the client files of the Nuxt build (`.nuxt/dist/client`) and the custom static files of the Nuxt app (`static`) with optimized cache settings.
- [Route53](https://aws.amazon.com/route53/): Two DNS records (`A` for IPv4 and `AAAA` for IPv6) in the configured hosted zone to make the Nuxt app available on the internet via the configured custom domain.
- [API Gateway](https://aws.amazon.com/api-gateway/): An HTTP API to make the Nuxt Lambda function publicly available.
- [CloudFront](https://aws.amazon.com/cloudfront/): A distribution to route incoming requests to the Nuxt Lambda function (via the API Gateway) and the S3 bucket to serve the static assets for the Nuxt app.
- [EventBridge](https://aws.amazon.com/eventbridge/): 
  - A scheduled rule to ping the Nuxt app's Lambda function every 5 minutes in order to keep it warm and to speed up initial SSR requests.
  - A scheduled rule to trigger the cleanup Lambda function for deleting the outdated static assets of the Nuxt app from S3 every tuesday at 03:30 AM GMT.