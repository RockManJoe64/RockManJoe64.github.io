---
layout: post
title: "Debugging C# AWS Lambdas"
date: "2021-06-08"
post_image: /assets/images/blog-img-6.jpg
tags: [csharp, aws, lambda]
categories: [programming]
author: jose
comments: true
dark_header: false
---

While working on a client project, I was distressed by the fact that debugging
the Lambda implementations was such a chore.

In order to debug a lambda, we would write a custom test harness around the
handler. The test harness would setup the environment variables and the event
object. On execution, the lambda would then connect to AWS resources via SSH
tunnels.

## Why It Sucked

Using the custom test harness wasn't bad. It was sufficient, but it took some
time to refine the test harnesses to a point they were easy to use. The
downside came with the fact that we had to create a unique harness for each
handler. Each harness was an NUnit test method marked with an Explicit
annotation. The annotation allowed the test runner to ignore them on unit test
executions.

The aggravating part came with making sure the harness would connect to the
external AWS services. In our case, we were leveraging Aurora DB, Document DB,
and SQS queues. It was a large project, so we had a dedicated DevOps team
managing those resources. But in order to access them, we would also need to
ensure that we had all our credentials up to date, and that we had a running
VPN tunnel to allow the connection to a bastion host. Any changes to the
services, their current state, or the credentials caused a block to our
development.

## Approach to Debugging

I decided to start from scratch. I created my own example application called [TraderNEXT](https://github.com/RockManJoe64/TraderNext), which mimics an equity order management system.

In starting from scratch, I used the
[AWS Toolkit for Visual Studio 2019](https://aws.amazon.com/visualstudio/)
to setup an example project. The toolkit will create an example lambda,
complete with a Dockerfile to be used to generate a container image. The
generated solution will also contain a `serverless.template` file,
which is really a YAML file using AWS Serverless configuration to tie your
lambdas to an API Gateway or other event source.

<figure class="figure">
<img
  src="{{site.baseurl}}/assets/images/screenshots/2021-06-08_screenshot1.png"
  alt="blog img"
/>
<figcaption class="figure-caption text-center">
  Creating a new Project in Visual Studio
</figcaption>
</figure>

The real benefits from using the toolkit comes in two ways:

- Launching the lambdas using the Visual Studio Debugger.
- Using the AWS SAM CLI tool to startup a running API to do integration
  testing.

## Visual Studio Debugger

When you use the debugger in Visual Studio, the toolkit will run a UI called
the Mock Lambda Test tool. This allows you to prepare API Gateway payloads to
test your lambdas with. Upon invoking the lambda, the Visual Studio debugger
will then stop at any breakpoints you've defined.

The advantage here is that you can do manual testing across all your lambdas
at the same time. There is no longer a need to maintain individual test
harnesses for each of the lambdas.

<figure class="figure">
<img
  src="{{site.baseurl}}/assets/images/screenshots/2021-06-08_screenshot2.png"
  alt="blog img"
/>
<figcaption class="figure-caption text-center">
  The Mock Lambda Test Tool
</figcaption>
</figure>

## Test Automation

Suppose after some rounds of testing, you've accumulated some constantly used
example payloads. You're getting tired of always having to manually copy/paste
the payloads into the mock tool. You can scratch that automation itch by
taking advantage of the AWS SAM CLI. There is some pre-work that needs to be
done, but its entirely possible to automate your tests against your lambdas
locally.
The setup I had to do was

1. First, create an AWS Elastic Container Registry (ECR) repository to upload a container image to. Any other container registry (GitHub, DockerHub) will also work.
1. Build, tag, and push an image of your lambdas to your repository.
1. Then, in your `serverless.template` file, you would define the image name using the ImageUri property.

I documented the steps to manually build, tag, and push a container image in a
shell script on my GitHub repo.
The following is the serverless definition for the GetOrders lambda:

<!-- prettier-ignore-start -->
{% highlight json %}
"GetOrders": {
  "Type": "AWS::Serverless::Function",
  "Properties": {
    "PackageType": "Image",
    "ImageConfig": {
      "EntryPoint": [
        "/lambda-entrypoint.sh"
      ],
      "Command": [
        "TraderNext::TraderNext.Lambdas.Orders.OrderLambdas::GetOrders"
      ]
    },
    "ImageUri": "public.ecr.aws/z8u1m2l7/tradernext:latest",
    "MemorySize": 256,
    "Timeout": 30,
    "Role": null,
    "Policies": [
      "AWSLambdaBasicExecutionRole"
    ],
    "Events": {
      "RootGet": {
        "Type": "Api",
        "Properties": {
          "Path": "/orders",
          "Method": "GET"
        }
      }
    }
  },
  "Metadata": {
    "Dockerfile": "Dockerfile",
    "DockerContext": ".",
    "DockerTag": ""
  }
}
{% endhighlight %}
<!-- prettier-ignore-end -->

Once that is done, you can launch your lambdas locally using this command:

<!-- prettier-ignore-start -->
{% highlight bash %}
sam start local-api
{% endhighlight %}
<!-- prettier-ignore-end -->

This will start a mock API Gateway instance on your machine, with all the
lambdas binded to specific URL paths defined in your serverless template. You
can then use any tool you wish to automate your testing. I used Postman to
automate this.

When you first invoke a lambda, you will see the SAM CLI download the related
image from your repository. This is why setting up the image and uploading it
first was necessary.

## Retrospective

The one thing that we cannot get away with is that the lambdas still need a
dependency on external data sources (databases, messaging queues). I've
created a docker compose file that will launch two containers for MySQL and
MongoDB. Both of these are great replacements for Aurora DB and Document DB. I
am using MySQL drivers to connect to Aurora DB anyway. The lambdas would
connect to these local resources when debugging or automated testing. You
would then need to setup your environment variables in the Mock Lambda tool to
ensure your lambdas connect correctly.

One way to circumvent this would be to just debug using your unit tests. Since
your dependencies would be mocked, the setup would be a lot simpler. But then
we would be moving back into the direction of a test harness!

## Conclusion

I believe by leveraging the tools AWS has provided, I've created something
much easier to debug with. I can debug my lambdas, as well as run automated
tests using only locally running resources. I've also setup the project to use
container images for easy deployment into AWS. I still have lots of work to do
on refining this process. I'll be sure to post more updates on this!
