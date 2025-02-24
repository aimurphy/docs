---
title: "Hot Reloading"
date: 2021-09-27
weight: 1
description: >
  Hot code reloading continuously applies code changes to Lambda functions
aliases:
  - /user-guide/tools/lambda-tools/hot-reloading/
---

Hot reloading (formerly known as hot swapping) continuously applies code changes to Lambda functions without manual redeployment.

Quickly iterating over Lambda function code can be quite cumbersome, as you need to deploy your function on every change.
LocalStack enables fast feedback cycles during development by automatically reloading your function code.
Pro users can also hot-reload Lambda layers.

{{< alert title="Note" >}}
**The magic S3 bucket name changed from `__local__` to `hot-reload` in LocalStack&nbsp;2.0**

Please change your deployment configuration accordingly because the old value is an invalid bucket name.
The configuration `BUCKET_MARKER_LOCAL` is still supported.

More information about the new Lambda provider is available under [Lambda providers]({{< ref "user-guide/aws/lambda" >}}).
{{< /alert >}}

## Covered Topics

* [Hot Reloading Behavior](#hot-reloading-behavior)
* [Application Configuration Examples](#application-configuration-examples):
  * [Hot reloading for JVM Lambdas](#hot-reloading-for-jvm-lambdas)
  * [Hot reloading for Python Lambdas](#hot-reloading-for-python-lambdas)
  * [Hot reloading for TypeScript Lambdas](#hot-reloading-for-typescript-lambdas)
* [Deployment Configuration Examples](#deployment-configuration-examples):
  * [Serverless Framework Configuration](#serverless-framework-configuration)
  * [AWS Cloud Development Kit (CDK) Configuration](#aws-cloud-development-kit-cdk-configuration)
  * [Terraform Configuration](#terraform-configuration)
* [Useful Links](#useful-links)

## Hot Reloading Behavior

**Delay in code change detection:**
It can take up to 700ms to detect code changes.
In the meantime, invocations still execute the former code.
Hot reloading triggers after 500ms without changes, and it can take up to an additional 200ms until the reloaded code is in effect.

**Runtime restart after each code change:**
The runtime inside the container is restarted after every code change.
During the runtime restart, the handler function re-executes any initialization code *outside* your handler function.
The container itself is not restarted.
Therefore, filesystem changes persist between code changes for invocations dispatched to the same container.

**File sharing permissions with Docker Desktop on macOS:**
If using Docker Desktop on macOS, you might need to allow [file sharing](https://docs.docker.com/desktop/settings/mac/#file-sharing) for your target folders.
MacOS may prompt you to grant Docker access to your target folders.

**Layer limit with hot reloading for layers:**
When hot reloading is active for a Lambda layer (Pro), the function can use at most one layer.

## Application Configuration Examples

### Hot reloading for JVM Lambdas

Since lambda containers lifetime is usually limited, regular hot code reloading
techniques are not applicable here.

In our implementation, we will be watching for fs changes under the project folder,
then build a `FatJar`, unzip it, and mount it into the Lambda Docker Container.

We assume you already have:
* [watchman](https://facebook.github.io/watchman/)
* configured JVM project capable of building FatJars using your preferred build tool

First, create a watchman wrapper by using
[one of our examples](https://github.com/localstack/localstack-pro-samples/tree/master/sample-archive/spring-cloud-function-microservice/bin/watchman.sh)

Don't forget to adjust permissions:
{{< command >}}
$ chmod +x bin/watchman.sh
{{< / command >}}

Now configure your build tool to unzip the FatJar to some folder, which will be
then mounted to LocalStack. We are using `Gradle` build tool to unpack the
`FatJar` into the `build/hot` folder:

```gradle
// We assume you are using something like `Shadow` plugin that comes with `shadowJar` task
task buildHot(type: Copy) {
    from zipTree("${project.buildDir}/libs/${project.name}-all.jar")
    into "${project.buildDir}/hot"
}
buildHot.dependsOn shadowJar
```

Now run the following command to start watching your project in a hot-reloading mode:

{{< command >}}
$ bin/watchman.sh src "./gradlew buildHot"
{{< / command >}}

Please note that you still need to configure your deployment tool to use
local code mounting. Read the [Deployment Configuration Examples](#deployment-configuration-examples)
for more information.

### Hot reloading for Python Lambdas

We will show you how you can do this with a simple example function, taken directly from the
[AWS Lambda developer guide](https://github.com/awsdocs/aws-doc-sdk-examples/blob/main/python/example_code/lambda/lambda_handler_basic.py).

You can check out that code, or use your own lambda functions to follow along. To use the example just do:

{{< command >}}
$ cd /tmp
$ git clone git@github.com:awsdocs/aws-doc-sdk-examples.git
{{< / command >}}

#### Creating the Lambda Function

To create the Lambda function, you just need to take care of two things:
1. Deploy via an S3 Bucket. You need to use the magic variable `hot-reload` as the bucket.
2. Set the S3 key to the path of the directory your lambda function resides in.
   The handler is then referenced by the filename of your lambda code and the function in that code that needs to be invoked.

So, using the AWS example, this would be:

{{< command >}}
$ awslocal lambda create-function --function-name my-cool-local-function \
    --code S3Bucket="hot-reload",S3Key="/tmp/aws-doc-sdk-examples/python/example_code/lambda" \
    --handler lambda_handler_basic.lambda_handler \
    --runtime python3.8 \
    --role arn:aws:iam::000000000000:role/lambda-role
{{< / command >}}

You can also check out some of our [Deployment Configuration Examples](#deployment-configuration-examples).

We can also quickly make sure that it works by invoking it with a simple payload:

{{< tabpane text=true persistLang=false >}}
{{% tab header="AWS CLI v1" lang="shell" %}}
{{< command >}}
$ awslocal lambda invoke --function-name my-cool-local-function \
    --payload '{"action": "increment", "number": 3}' \
    output.txt
{{< /command >}}
{{% /tab %}}
{{% tab header="AWS CLI v2" lang="shell" %}}
{{< command >}}
$ awslocal lambda invoke --function-name my-cool-local-function \
    --cli-binary-format raw-in-base64-out \
    --payload '{"action": "increment", "number": 3}' \
    output.txt
{{< /command >}}
{{% /tab %}}
{{< /tabpane >}}

The invocation returns itself returns:

```json
{
    "StatusCode": 200,
    "LogResult": "",
    "ExecutedVersion": "$LATEST"
}
```
and `output.txt` contains:
```json
{"result":4}
```

#### Changing things up

Now, that we got everything up and running, the fun begins.
Because the function is now mounted as a file in the executing container, any change that we save on the file will be there in an instant.

For example, we can now make a minor change to the API and replace the response in [line 41](https://github.com/awsdocs/aws-doc-sdk-examples/blob/main/python/example_code/lambda/lambda_handler_basic.py#L41) with the following:

```python
    response = {'math_result': result}
```

Without redeploying or updating the function, the result of the previous request will look like this:

```json
{"math_result":4}
```
Cool!

#### Usage with Virtualenv

For [virtualenv](https://virtualenv.pypa.io)-driven projects, all dependencies should be made
available to the Python interpreter at runtime. There are different ways to achieve that, including:
* expanding the Python module search path in your Lambda handler
* creating a watchman script to copy the libraries

##### Expanding the module search path in your Lambda handler

The easiest approach is to expand the module search path (`sys.path`) and add the `site-packages` folder inside the virtualenv.
We can add the following two lines of code at the top of the Lambda handler script:

```python
import sys, glob
sys.path.insert(0, glob.glob(".venv/lib/python*/site-packages")[0])
...
import some_lib_from_virtualenv  # import your own modules here
```

This way you can easily import modules from your virtualenv, without having to change the file system layout.

Note: As an alternative to modifying `sys.path`, you could also set the `PYTHONPATH` environment variable when creating your Lambda function, to add the additional path.

##### Using a watchman script to copy libraries

Another alternative is to implement a watchman script that will be preparing a special folder for hot code reloading.

In our example, we are using `build/hot` folder as a mounting point for our Lambdas.

First, create a watchman wrapper by using
[one of our examples](https://github.com/localstack/localstack-pro-samples/tree/master/sample-archive/spring-cloud-function-microservice/bin/watchman.sh)

After that, you can use the following `Makefile` snippet, or implement another shell script to prepare the codebase for hot reloading:

```make
BUILD_FOLDER ?= build
PROJECT_MODULE_NAME = my_project_module

build-hot:
	rm -rf $(BUILD_FOLDER)/hot && mkdir -p $(BUILD_FOLDER)/hot
	cp -r $(VENV_DIR)/lib/python$(shell python --version | grep -oE '[0-9]\.[0-9]')/site-packages/* $(BUILD_FOLDER)/hot/
	cp -r $(PROJECT_MODULE_NAME) $(BUILD_FOLDER)/hot/$(PROJECT_MODULE_NAME)
	cp *.toml $(BUILD_FOLDER)/hot

watch:
	bin/watchman.sh $(PROJECT_MODULE_NAME) "make build-hot"

.PHONY: build-hot watch
```

To run the example above, run `make watch`. The script is copying the project module `PROJECT_MODULE_NAME`
along with all dependencies into the `build/hot` folder, which is then mounted into
LocalStack's Lambda container.

### Hot reloading for TypeScript Lambdas

You can hot-reload your [TypeScript Lambda functions](https://docs.aws.amazon.com/lambda/latest/dg/lambda-typescript.html). We will check-out a simple example to create a simple `Hello World!` Lambda function using TypeScript.

#### Setting up the Lambda function

Create a new Node.js project with `npm` or an alternative package manager:

{{< command >}}
$ npm init -y
{{< / command >}}

Install the the [@types/aws-lambda](https://www.npmjs.com/package/@types/aws-lambda) and [esbuild](https://esbuild.github.io/) packages in your Node.js project:

{{< command >}}
$ npm install -D @types/aws-lambda esbuild
{{< / command >}}

Create a new file named `index.ts`. Add the following code to the new file:

```ts
import { Context, APIGatewayProxyResult, APIGatewayEvent } from 'aws-lambda';

export const handler = async (event: APIGatewayEvent, context: Context): Promise<APIGatewayProxyResult> => {
  console.log(`Event: ${JSON.stringify(event, null, 2)}`);
  console.log(`Context: ${JSON.stringify(context, null, 2)}`);
  return {
      statusCode: 200,
      body: JSON.stringify({
          message: 'Hello World!',
      }),
   };
};
```

Add a build script to your `package.json` file:

```json
"scripts": {
    "build": "esbuild index.ts --bundle --minify --sourcemap --platform=node --target=es2020 --outfile=dist/index.js --watch"
  },
```

The build script will use `esbuild` to bundle and minify the TypeScript code into a single JavaScript file, which will be placed in the `dist` folder.

You can now run the build script to create the `dist/index.js` file:

{{< command >}}
$ npm run build
{{< / command >}}

#### Creating the Lambda Function

To create the Lambda function, you need to take care of two things:

- Deploy via an S3 Bucket. You need to use the magic variable `hot-reload` as the bucket.
- Set the S3 key to the path of the directory your lambda function resides in. The handler is then referenced by the filename of your lambda code and the function in that code that needs to be invoked.

Create the Lambda Function using the `awslocal` CLI:

{{< command >}}
awslocal lambda create-function \
    --function-name hello-world \
    --runtime "nodejs16.x" \
    --role arn:aws:iam::123456789012:role/lambda-ex \
    --code S3Bucket="hot-reload",S3Key="$(PWD)/dist" \
    --handler index.handler
{{< / command >}}

You can quickly make sure that it works by invoking it with a simple payload:

{{< tabpane text=true persistLang=false >}}
{{% tab header="AWS CLI v1" lang="shell" %}}
{{< command >}}
$ awslocal lambda invoke \
    --function-name hello-world \
    --payload '{"action": "test"}' \
    output.txt
{{< /command >}}
{{% /tab %}}
{{% tab header="AWS CLI v2" lang="shell" %}}
{{< command >}}
$ awslocal lambda invoke \
    --function-name hello-world \
    --cli-binary-format raw-in-base64-out \
    --payload '{"action": "test"}' \
    output.txt
{{< /command >}}
{{% /tab %}}
{{< /tabpane >}}

The invocation returns itself returns:

```sh
{
    "StatusCode": 200,
    "ExecutedVersion": "$LATEST"
}
```

The `output.txt` file contains the following:

```sh
{"statusCode":200,"body":"{\"message\":\"Hello World!\"}"}
```

#### Changing the Lambda Function

The Lambda function is now mounted as a file in the executing container, hence any change that we save on the file will be there in an instant.

Change the `Hello World!` message to `Hello LocalStack!` and run `npm run build`. Trigger the Lambda once again. You will see the following in the `output.txt` file:

```sh
{"statusCode":200,"body":"{\"message\":\"Hello LocalStack!\"}"}
```

## Deployment Configuration Examples

### Serverless Framework Configuration

Enable local code mounting

```yaml
custom:
  localstack:
    ...
    lambda:
      mountCode: true

# or if you need to enable code mounting only for specific stages
custom:
  stages:
    local:
      mountCode: true
    testing:
      mountCode: false
  localstack:
    stages:
      - local
      - testing
    lambda:
      mountCode: ${self:custom.stages.${opt:stage}.mountCode}
```

Pass `LAMBDA_MOUNT_CWD` env var with path to the built code directory
(in our case to the folder with unzipped FatJar):

{{< command >}}
$ LAMBDA_MOUNT_CWD=$(pwd)/build/hot serverless deploy --stage local
{{< / command >}}

### AWS Cloud Development Kit (CDK) Configuration

{{< highlight kotlin "linenos=table" >}}
package org.localstack.cdkstack

import java.util.UUID
import software.amazon.awscdk.core.Construct
import software.amazon.awscdk.core.Duration
import software.amazon.awscdk.core.Stack
import software.amazon.awscdk.services.lambda.*
import software.amazon.awscdk.services.s3.Bucket

private val STAGE = System.getenv("STAGE") ?: "local"
private val LAMBDA_MOUNT_CWD = System.getenv("LAMBDA_MOUNT_CWD") ?: ""
private const val JAR_PATH = "build/libs/localstack-sampleproject-all.jar"

class ApplicationStack(parent: Construct, name: String) : Stack(parent, name) {

    init {
        val lambdaCodeSource = this.buildCodeSource()

        SingletonFunction.Builder.create(this, "ExampleFunctionOne")
            .code(lambdaCodeSource)
            .handler("org.localstack.sampleproject.api.LambdaApi")
            .environment(mapOf("FUNCTION_NAME" to "functionOne"))
            .timeout(Duration.seconds(30))
            .runtime(Runtime.JAVA_11)
            .uuid(UUID.randomUUID().toString())
            .build()
    }

    /**
     * Mount code for hot-reloading when STAGE=local
     */
    private fun buildCodeSource(): Code  {
        if (STAGE == "local") {
            val bucket = Bucket.fromBucketName(this, "HotReloadingBucket", "hot-reload")
            return Code.fromBucket(bucket, LAMBDA_MOUNT_CWD)
        }

        return Code.fromAsset(JAR_PATH)
    }
}
{{< / highlight >}}

Then to bootstrap and deploy the stack run the following shell script

{{< command >}}
$ STAGE=local && LAMBDA_MOUNT_CWD=$(pwd)/build/hot &&
  cdklocal bootstrap aws://000000000000/$(AWS_REGION) && \
  cdklocal deploy
{{< / command >}}

### Terraform Configuration

```hcl
variable "STAGE" {
    type    = string
    default = "local"
}

variable "AWS_REGION" {
    type    = string
    default = "us-east-1"
}

variable "JAR_PATH" {
    type    = string
    default = "build/libs/localstack-sampleproject-all.jar"
}

variable "LAMBDA_MOUNT_CWD" {
    type    = string
}

provider "aws" {
    access_key                  = "test_access_key"
    secret_key                  = "test_secret_key"
    region                      = var.AWS_REGION
    s3_force_path_style         = true
    skip_credentials_validation = true
    skip_metadata_api_check     = true
    skip_requesting_account_id  = true

    endpoints {
        apigateway       = var.STAGE == "local" ? "http://localhost:4566" : null
        cloudformation   = var.STAGE == "local" ? "http://localhost:4566" : null
        cloudwatch       = var.STAGE == "local" ? "http://localhost:4566" : null
        cloudwatchevents = var.STAGE == "local" ? "http://localhost:4566" : null
        iam              = var.STAGE == "local" ? "http://localhost:4566" : null
        lambda           = var.STAGE == "local" ? "http://localhost:4566" : null
        s3               = var.STAGE == "local" ? "http://localhost:4566" : null
    }
}

resource "aws_iam_role" "lambda-execution-role" {
    name = "lambda-execution-role"

    assume_role_policy = <<EOF
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Action": "sts:AssumeRole",
      "Principal": {
        "Service": "lambda.amazonaws.com"
      },
      "Effect": "Allow",
      "Sid": ""
    }
  ]
}
EOF
}

resource "aws_lambda_function" "exampleFunctionOne" {
    s3_bucket     = var.STAGE == "local" ? "hot-reload" : null
    s3_key        = var.STAGE == "local" ? var.LAMBDA_MOUNT_CWD : null
    filename      = var.STAGE == "local" ? null : var.JAR_PATH
    function_name = "ExampleFunctionOne"
    role          = aws_iam_role.lambda-execution-role.arn
    handler       = "org.localstack.sampleproject.api.LambdaApi"
    runtime       = "java11"
    timeout       = 30
    source_code_hash = filebase64sha256(var.JAR_PATH)
    environment {
        variables = {
            FUNCTION_NAME = "functionOne"
        }
    }
}
```

{{< command >}}
$ terraform init && \
  terraform apply -var "STAGE=local" -var "LAMBDA_MOUNT_CWD=$(pwd)/build/hot"
{{< / command >}}

## Useful Links

* [Lambda Code Mounting and Debugging (Python)](https://github.com/localstack/localstack-pro-samples/tree/master/lambda-mounting-and-debugging)
* [Spring Cloud Function on LocalStack (Kotlin JVM)](https://github.com/localstack/localstack-pro-samples/tree/master/sample-archive/spring-cloud-function-microservice)
