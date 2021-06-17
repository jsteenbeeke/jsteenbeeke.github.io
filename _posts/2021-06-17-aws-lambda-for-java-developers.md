---
layout: post
title:  AWS Lambda for Java developers
author: "Jeroen Steenbeeke"
date:   2021-06-17 17:45:00 +0200
---
Serverless programming is something I've known about for quite some time, but never really gotten around
to doing anything with. Finally deciding to see what the fuss was about, I set out to build something
that could be run on AWS Lambda. This was surprisingly easy, and in this article I will illustrate how
to create a simple function that converts a `long` representing seconds since the Unix epoch to an ISO-8601 formatted date.

<!--more-->

## Getting started

Maven is my go-to tool for managing projects, and to develop a lambda function you need to create 
simple project and add the following dependency:

```xml
<dependency>
	<groupId>com.amazonaws</groupId>
	<artifactId>aws-lambda-java-core</artifactId>
    <version>${amazon.aws.lambda.java.core.version}</version>
</dependency>
```
This requires you to set a property for the latest SDK version. I used version 1.2.1:

```xml
    <properties>
        <maven.compiler.source>11</maven.compiler.source>
        <maven.compiler.target>11</maven.compiler.target>
        <amazon.aws.lambda.java.core.version>1.2.1</amazon.aws.lambda.java.core.version>
    </properties>
```
AWS Lambda currently supports Java versions 8 and 11. With all this configured we can now create our handler.

## Function handler

The function handler is a class that implements the `RequestHandler` interface:

```java
public class ISODateFormatter implements RequestHandler<Long,String> {
	@Override
	public String handleRequest(Long seconds, Context context)
	{
		LocalDateTime instant = LocalDateTime.ofEpochSecond(seconds, 0, ZoneOffset.UTC);

		return DateTimeFormatter.ISO_LOCAL_DATE_TIME.format(instant);
	}
}
```
And that's basically it. AWS Lambda is pretty flexible in the sort of inputs and outputs it supports.
You can easily have it receive and return JSON objects.

## Deploying the function

To deploy the function, we need to do several things. First we need to compile our function and copy any dependencies
it includes to our target folder. Next, we need to package all this in a Docker container that uses
Amazon's Java images as a base. Then we need to deploy the image to a registry that AWS can read from,
such as an Amazon ECR registry, and finally we need to create the actual function in AWS Lambda

### Configuring Maven to include dependencies



### Creating the Docker image

We need to use one of Amazon's base images to package our Lambda function:

```dockerfile
# Use Amazon's Java 11 base image
FROM public.ecr.aws/lambda/java:11

# Copy all classes to the Java root of the image
COPY target/classes ${LAMBDA_TASK_ROOT}

# Copy all dependency libraries to the library folder
COPY target/dependency/* ${LAMBDA_TASK_ROOT}/lib/

# Set the handler class and method as the command to execute
CMD [ "nl.jeroensteenbeeke.tech.lambda.isodate.ISODateFormatter::handleRequest" ]
```

This can easily be built using a standard `docker build .` command, but to get this image deployed
it's best to follow the instructions on Amazon ECR.

For now, for testing purposes, we can do the following:

```shell
docker build -t isodate-example:latest .
docker run -p 9000:8080 isodate-example:latest
```

Then from another window/tab we can use `curl` to send a request:

```shell
$ curl -XPOST "http://localhost:9000/2015-03-31/functions/function/invocations" -d '1623923209'
"2021-06-17T09:46:49"
```

This way, we can test the lambda function before deploying it on Amazon and paying for it.

### Deploying the Docker image

To deploy the Docker image, you need to set up an Amazon Elastic Container Registry (ECR) repository for your image. Pick a suitable
name for your repository (I used `isodate-example` for this one), and then from the ECR page select
your repository, and click "View Push Commands". This will give you instructions on how to push your
image.

Doing so requires that you have the `aws` command line utility installed, and that you have access to
the registry with the credentials configured for this command.

**I strongly recommend creating a separate access key just for registry access**. This can be configured
using Amazon IAM, and [Amazon has good documentation on this subject](https://docs.aws.amazon.com/AmazonECR/latest/userguide/Registries.html#registry_auth).

### Creating the function

Creating the function is simply a matter of going to the AWS Lambda interface, giving it a name,
and selecting an image using the "Browse images" functionality. Amazon will also give you the option
of creating a role specifically for this function, but this does not configure access to the Lambda function.

A policy such as the following will suffice (remember to input the ARN of your function):

```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Sid": "VisualEditor0",
            "Effect": "Allow",
            "Action": [
                "lambda:InvokeFunction",
                "lambda:InvokeAsync"
            ],
            "Resource": [
                "ARN-OF-YOUR-LAMBDA-GOES-HERE"
            ]
        }
    ]
}
```

When creating an API user to invoke the lambda, remember to write down the secret before you click okay,
there is no way of retrieving it once your close the page.

## Using the function

To use the function, you need to use the Amazon SDK. First we need to add the SDK dependency:

```xml
    <dependency>
        <groupId>com.amazonaws</groupId>
        <artifactId>aws-java-sdk-lambda</artifactId>
        <version>${amazon.aws.lambda.sdk.version}</version>
    </dependency>
```
I used version `1.12.2`:
```xml
<properties>
	<amazon.aws.lambda.sdk.version>1.12.2</amazon.aws.lambda.sdk.version>
</properties>
```
To use the SDK, you first need to create the client:
```java
public AWSLambda createClient(String key, String secret, String region) {
	return AWSLambdaClientBuilder.standard()
		.withCredentials(new AWSStaticCredentialsProvider(new BasicAWSCredentials(key,secret))
		.withRegion(region)
		.build();
}
```
You can then use this client to invoke any function you have access to. Let's try the function we
wrote earlier. Keep in mind that all inputs and outputs are formatted in JSON:
```java
// Use Jackson to convert the output
ObjectMapper objectMapper = new ObjectMapper();
String arn = "MY_ARN";

InvokeResult invokeResult = lambda.invoke(new InvokeRequest()
                                  .withFunctionName(arn)
                                  .withPayload("5"));

if (invokeResult.getStatusCode() == 200) {
	String result = objectMapper.readValue(invokeResult.getPayload().array(), String.class);
	System.out.printf("Result    : %s%n", result);	
} else {
	// Invocation failed
	System.err.printf("Error     : %s%n", invokeResult.getFunctionError());
	System.err.printf("Log result: %s%n", invokeResult.getLogResult());    
}
```
If the permissions are configured correctly, this will output:

> Result    : 1970-01-01T00:00:05

## In conclusion

This is just a simple example of a direct invocation of an AWS Lambda function, but we've barely
scratched the surface. AWS Lambda can be configured to respond to events, and Lambda functions can
be chained to achieve more complex functionality.

To check this out for yourself you can check [the code I used for this blog post on Github](https://github.com/jsteenbeeke/aws-lambda-blog-post).
