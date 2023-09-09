---
layout: post
title: A service renewal of backend built upon AWS Lambda
date: 2021-02-17T02:51:11+0300
description: 
tags: TypeScript NestJS AWS Datadog
categories: Backend
giscus_comments: true
related_posts: false
toc:
  sidebar: left
---

I recently worked on [38qa.net](https://38qa.net/) migration as a backend developer. The current service is one monolithic app based on [question2answer](https://github.com/q2a/question2answer) which is an open source Q&A platform for PHP/MySQL. (The new one hasn’t been released yet as of February, 2021.) We divided the app into frontend and backend written in TypeScript but still kept the same feature it used to have. On this blog post, I would like to look back the backend development.

## Motivation

The current app uses a monolithic framework called [question2answer](https://github.com/q2a/question2answer). The framework has been barely maintained but there are lots of legacy code in it. Besides, the old app heavily depends on the extension [plugins](https://docs.question2answer.org/plugins/) of the framework. That makes the code more difficult to make new changes and scale the project. To run the service for another decade, the business owner wants to pay off the technical debt.

## Tech Stack

### TypeScript

38qa.net is owned by a small company called [weekend bee-keeping](https://syumatsu-yoho.co.jp/) (週末養蜂). As of 2021, it is easier for them to find a TypeScript and React developer rather than a PHP developer since the company hires freelance developers. Also, it is easier for the developers to read and write both frontend and backend code in the same programming language. The common code can be shared on both sides as well. Therefore, the owner decided to use TypeScript.

### NestJS

The frontend framework was already fixed before I joined the development since [React](https://github.com/facebook/react) is the recent main stream. I looked for a backend framework. Then I found [NestJS](https://github.com/nestjs/nest). NestJS provides Rails-Like intuitive project structure, testing, and command-line tools. [Rails](https://github.com/rails/rails)-Like framework would be a good fit for a small company like weekend bee-keeping due to the balance between good maintainability as long as you ride on the rail and high speed development especially within small number of teammates. Moreover, NestJS has a huge and active developer’s community. Here is a [nestjs sample project](https://github.com/kenzan8000/nestjs-lambda-boilerplate).

### Fastify

NestJS makes use of Express under the hood but provides compatibility with a wide range of other servers like [Fastify](https://github.com/fastify/fastify). Fastify would be a faster and lower overhead server than Express according to the [benchmark](https://github.com/fastify/fastify#benchmarks). In addition, the developer’s community is huge and active, too.

## Architecture

<div class="row">
    <div class="col-sm mt-3 mt-md-0">
        {% include figure.html path="/assets/img/2021-02-17-a-service-renewal-of-backend-built-upon-aws-lambda-architecture.jpg" title="Architecture" class="img-fluid rounded z-depth-1" %}
    </div>
</div>

### Serverless + AWS Lambda

The current app was originally hosted on a single [AWS EC2](https://aws.amazon.com/ec2) instance. The business owner was interested in the architecture built by [Serverless](https://github.com/serverless/serverless) on [AWS Lambda](https://aws.amazon.com/lambda/) since he wanted to use a modern technology and save the server cost. I actually imagined the new architecture would be on [AWS ECS](https://aws.amazon.com/ecs) or [EKS](https://aws.amazon.com/eks/) but I didn’t stick to any specific one. Why don’t we try Lambda out? He and I decided to take Lambda first but if it’s hard, we move on to another architecture.

### Autoscaling

AWS Lambda is pretty easy to use because it already provides automated scaling and complete elimination of container maintenance. However, there was a problem. If none of your Lambda functions have been invoked in some time, it takes time until your application code is executed. The slow launching problem is called Cold Start. Fortunately, [AWS announced a solution called Provisioned Concurrency at the end of 2019](https://aws.amazon.com/about-aws/whats-new/2019/12/aws-lambda-announces-provisioned-concurrency).

With Provisioned Concurrency, functions can instantaneously serve a burst of traffic with consistent start-up latency for every invoke up to the specified scale.

Thank the developer’s community and the great [Serverless plugin](https://web.archive.org/web/20230203031220/https://medium.com/neiman-marcus-tech/serverless-provisioned-concurrency-autoscaling-3d8ec23d10c).

### MySQL

We continue to use the same MySQL database the current app uses but there was the following problem.

Many applications, including those built on modern serverless architectures, can have a large number of open connections to the database server, and may open and close database connections at a high rate, exhausting database memory and compute resources.

Fortunately, [AWS announced a solution called RDS Proxy on 2020](https://aws.amazon.com/blogs/aws/amazon-rds-proxy-now-generally-available).

Amazon RDS Proxy allows applications to pool and share connections established with the database, improving database efficiency and application scalability.

### Cognito

We use [Amazon Cognito](https://aws.amazon.com/cognito) to handle the user authentication.

Amazon Cognito lets you add user sign-up, sign-in, and access control to your web and mobile apps quickly and easily. Amazon Cognito scales to millions of users and supports sign-in with social identity providers, such as Facebook, Google, and Amazon, and enterprise identity providers via SAML 2.0.

The current app already has the own implementation of the user authentication. However, the implementation is a bit legacy. On the other hand, Cognito is a well-designed system maintained by Amazon. Also, we expect the synergetic effect because most of 38qa.net tech stack is on AWS. For example, [Amazon SES](https://aws.amazon.com/ses) can send the email to a user when the user signs in for the first time. This kind of the integration is easily done on AWS Management Console.

To migrate the app users to Amazon Cognito User Pools, [Cognito prepares some options](https://web.archive.org/web/20230606003242/https://aws.amazon.com/blogs/mobile/migrating-users-to-amazon-cognito-user-pools/). One option is one-at-a-time user migration.

The one-at-a-time user migration method involves first attempting to sign in the user through the Amazon Cognito User Pool. Then, if that sign-in fails, you sign them in through the existing user directory and capture the user name and password to silently create the user in the user pool.

Cognito invokes Lambda function for the first attempt when a user signs in but the user doesn’t exist in the user pool yet. The data for the migration is exchanged between Cognito and your Lambda function through the Webhook-Like procedure below.

<div class="row">
    <div class="col-sm mt-3 mt-md-0">
        {% include figure.html path="/assets/img/2021-02-17-a-service-renewal-of-backend-built-upon-aws-lambda-cognito.jpg" title="Cognito" class="img-fluid rounded z-depth-1" %}
    </div>
</div>

### Cron Job

The new 38qa.net cron job can be classified as three types. They are executed on Lambda function.

The first one is a scheduler running jobs on your app every scheduled time interval. For example, ranking table is updated every hour.

Second one is an event triggered by a certain condition. For example, when an image is uploaded, the cron job generates a thumbnail image from the original image.

The other one is one-time job that runs only once. For example, the current app had binary image data in the db column. The cron job uploads the image to the storage.

<div class="row">
    <div class="col-sm mt-3 mt-md-0">
        {% include figure.html path="/assets/img/2021-02-17-a-service-renewal-of-backend-built-upon-aws-lambda-cron.jpg" title="Cron" class="img-fluid rounded z-depth-1" %}
    </div>
</div>

### Datadog

<div class="row">
    <div class="col-sm mt-3 mt-md-0">
        {% include figure.html path="/assets/img/2021-02-17-a-service-renewal-of-backend-built-upon-aws-lambda-datadog-dashboard.jpg" title="Datadog dashboard" class="img-fluid rounded z-depth-1" %}
    </div>
</div>

The new 38qa.net uses [Dabtadog](https://www.datadoghq.com/) for the backend monitoring. Datadog monitors the useful metrics to optimize serverless function such as errors and cold starts, allocated memory, environment, etc. It identifies high-level function issues, and provides the intuitive traces and logs on the dashboard.

Your Lambda metrics go to CloudWatch via Datadog Lambda layer first. Then another Lambda function called Datadog Forwarder subscribes the CloudWatch updates and pushes them into Datadog.

<div class="row">
    <div class="col-sm mt-3 mt-md-0">
        {% include figure.html path="/assets/img/2021-02-17-a-service-renewal-of-backend-built-upon-aws-lambda-datadog.jpg" title="Datadog" class="img-fluid rounded z-depth-1" %}
    </div>
</div>

### Load Testing

<div class="row">
    <div class="col-sm mt-3 mt-md-0">
        {% include figure.html path="/assets/img/2021-02-17-a-service-renewal-of-backend-built-upon-aws-lambda-artillery.jpg" title="Artillery" class="img-fluid rounded z-depth-1" %}
    </div>
</div>

To run the load testing, I decided to use [artillery](https://artillery.io/). Artillery allows you to make performance and functionality testing quickly, easily and without having to maintain any servers or testing infrastructure. I also used [serverless-artillery](https://github.com/Nordstrom/serverless-artillery) plugin that helps you easily install the load testing environment on Lambda and [artillery-plugin-datadog](https://www.npmjs.com/package/artillery-plugin-datadog) plugin that submits Artillery metrics to Datadog over HTTPS.

<div class="row">
    <div class="col-sm mt-3 mt-md-0">
        {% include figure.html path="/assets/img/2021-02-17-a-service-renewal-of-backend-built-upon-aws-lambda-artillery-on-datadog-dashboard.jpg" title="Artillery on Datadog dashboard" class="img-fluid rounded z-depth-1" %}
    </div>
</div>

By the way, for the rate limiting, I used [nestjs-rate-limiter](https://github.com/ozkanonur/nestjs-rate-limiter) which adds in configurable rate limiting.

## Conclusion

I firstly doubted if it is possible to build a fully functioning system on AWS Lambda. There are still some problems that aren’t perfectly solved yet. i.e. Cold Start problem. However, as a result, the recent AWS updates enable you to build most of the architecture on AWS Lambda.