---
layout: post
title: A service renewal of a backend built on AWS Lambda
date: 2021-02-17T02:51:11+0300
description: I recently worked on a website backend migration. This post explains how we approached it.
tags: TypeScript NestJS AWS Datadog
categories: Backend
giscus_comments: true
related_posts: false
toc:
  sidebar: left
---

I recently worked on the [38qa.net](https://38qa.net/) migration as a backend developer. At the time, 38qa.net was a monolithic app built on [question2answer](https://github.com/q2a/question2answer), an open-source Q&A platform for PHP/MySQL. We split the app into a TypeScript frontend and backend, while keeping the same features. The new version has since been released. This post looks back at the backend work.

## Motivation

The old app used a monolithic framework called [question2answer](https://github.com/q2a/question2answer). The framework was barely maintained, and it contained a lot of legacy code. The old app also depended heavily on the framework's extension [plugins](https://docs.question2answer.org/plugins/), which made new changes and scaling harder. To keep the service running for another decade, the business owner wanted to pay off that technical debt.

## Tech Stack

### TypeScript

38qa.net is owned by a small company called [weekend bee-keeping](https://syumatsu-yoho.co.jp/) (週末養蜂). As of 2021, it is easier for them to find a TypeScript and React developer than a PHP developer, since the company hires freelancers. It is also easier for developers to read and write frontend and backend code in the same language, and to share common code between the two. That's why the owner chose TypeScript.

### NestJS

The frontend framework was already decided before I joined: [React](https://github.com/facebook/react) is the current mainstream choice. I looked for a backend framework and found [NestJS](https://github.com/nestjs/nest). NestJS gives you a Rails-like project structure, testing, and command-line tools. A [Rails](https://github.com/rails/rails)-like framework is a good fit for a small company like weekend bee-keeping: you get maintainability as long as you stay on the rails, and you can move quickly with a small team. NestJS also has a large, active community. Here is a [NestJS sample project](https://github.com/kenzan8000/nestjs-lambda-boilerplate).

### Fastify

NestJS uses Express under the hood, but it also works with other servers such as [Fastify](https://github.com/fastify/fastify). Fastify is faster and has lower overhead than Express, according to the [benchmark](https://github.com/fastify/fastify#benchmarks). It has a large, active community as well.

## Architecture

<div class="row">
    <div class="col-sm mt-3 mt-md-0">
        {% include figure.liquid path="/assets/img/2021-02-17-a-service-renewal-of-backend-built-upon-aws-lambda-architecture.jpg" title="Architecture" class="img-fluid rounded z-depth-1" %}
    </div>
</div>

### Serverless + AWS Lambda

The old app originally ran on a single [AWS EC2](https://aws.amazon.com/ec2) instance. The business owner was interested in an architecture built with [Serverless](https://github.com/serverless/serverless) on [AWS Lambda](https://aws.amazon.com/lambda/), both to use a more modern stack and to save on server cost. I had imagined the new system on [AWS ECS](https://aws.amazon.com/ecs) or [EKS](https://aws.amazon.com/eks/), but I didn't insist on either. Why not try Lambda first? We agreed to start with Lambda, and move to another architecture if it proved too hard.

### Autoscaling

AWS Lambda is easy to use because it already handles automated scaling and removes container maintenance. There was one problem, though. If none of your Lambda functions have been invoked for a while, it takes time before your application code runs. That slow launch is a cold start. Fortunately, [AWS announced Provisioned Concurrency at the end of 2019](https://aws.amazon.com/about-aws/whats-new/2019/12/aws-lambda-announces-provisioned-concurrency).

> With Provisioned Concurrency, functions can instantaneously serve a burst of traffic with consistent start-up latency for every invoke up to the specified scale.

Thanks to the developer community and this [Serverless plugin](https://web.archive.org/web/20230203031220/https://medium.com/neiman-marcus-tech/serverless-provisioned-concurrency-autoscaling-3d8ec23d10c).

### MySQL

We kept using the same MySQL database as the old app, but we ran into the following problem.

> Many applications, including those built on modern serverless architectures, can have a large number of open connections to the database server, and may open and close database connections at a high rate, exhausting database memory and compute resources.

Fortunately, [AWS announced RDS Proxy in 2020](https://aws.amazon.com/blogs/aws/amazon-rds-proxy-now-generally-available).

> Amazon RDS Proxy allows applications to pool and share connections established with the database, improving database efficiency and application scalability.

### Cognito

We use [Amazon Cognito](https://aws.amazon.com/cognito) for user authentication.

> Amazon Cognito lets you add user sign-up, sign-in, and access control to your web and mobile apps quickly and easily. Amazon Cognito scales to millions of users and supports sign-in with social identity providers, such as Facebook, Google, and Amazon, and enterprise identity providers via SAML 2.0.

The old app already had its own authentication, but that implementation was a bit dated. Cognito is a well-designed system maintained by Amazon. We also expected a compounding benefit because most of the 38qa.net stack is already on AWS. For example, [Amazon SES](https://aws.amazon.com/ses) can email a user when they sign in for the first time. That kind of integration is easy to set up in the AWS Management Console.

To migrate existing users into Amazon Cognito User Pools, [Cognito offers a few options](https://web.archive.org/web/20230606003242/https://aws.amazon.com/blogs/mobile/migrating-users-to-amazon-cognito-user-pools/). One of them is one-at-a-time user migration.

> The one-at-a-time user migration method involves first attempting to sign in the user through the Amazon Cognito User Pool. Then, if that sign-in fails, you sign them in through the existing user directory and capture the user name and password to silently create the user in the user pool.

Cognito invokes a Lambda function on the first attempt when a user signs in but does not yet exist in the user pool. The migration data is exchanged between Cognito and that Lambda function through the webhook-like flow below.

<div class="row">
    <div class="col-sm mt-3 mt-md-0">
        {% include figure.liquid path="/assets/img/2021-02-17-a-service-renewal-of-backend-built-upon-aws-lambda-cognito.jpg" title="Cognito" class="img-fluid rounded z-depth-1" %}
    </div>
</div>

### Cron Job

The new 38qa.net jobs fall into three types. All of them run on Lambda.

The first is a scheduler that runs jobs at a set interval. For example, the ranking table is updated every hour.

The second is an event triggered by a condition. For example, when an image is uploaded, a job generates a thumbnail from the original.

The third is a one-time job. For example, the old app stored binary image data in a database column. A one-time job uploads those images to storage.

<div class="row">
    <div class="col-sm mt-3 mt-md-0">
        {% include figure.liquid path="/assets/img/2021-02-17-a-service-renewal-of-backend-built-upon-aws-lambda-cron.jpg" title="Cron" class="img-fluid rounded z-depth-1" %}
    </div>
</div>

### Datadog

<div class="row">
    <div class="col-sm mt-3 mt-md-0">
        {% include figure.liquid path="/assets/img/2021-02-17-a-service-renewal-of-backend-built-upon-aws-lambda-datadog-dashboard.jpg" title="Datadog dashboard" class="img-fluid rounded z-depth-1" %}
    </div>
</div>

The new 38qa.net uses [Datadog](https://www.datadoghq.com/) for backend monitoring. Datadog tracks metrics that help optimize serverless functions, such as errors, cold starts, allocated memory, and environment. It flags high-level function issues and shows traces and logs on the dashboard.

Lambda metrics go to CloudWatch through the Datadog Lambda layer first. Then another Lambda function, Datadog Forwarder, subscribes to those CloudWatch updates and pushes them into Datadog.

<div class="row">
    <div class="col-sm mt-3 mt-md-0">
        {% include figure.liquid path="/assets/img/2021-02-17-a-service-renewal-of-backend-built-upon-aws-lambda-datadog.jpg" title="Datadog" class="img-fluid rounded z-depth-1" %}
    </div>
</div>

### Load Testing

<div class="row">
    <div class="col-sm mt-3 mt-md-0">
        {% include figure.liquid path="/assets/img/2021-02-17-a-service-renewal-of-backend-built-upon-aws-lambda-artillery.jpg" title="Artillery" class="img-fluid rounded z-depth-1" %}
    </div>
</div>

For load testing, I used [Artillery](https://artillery.io/). Artillery makes it straightforward to run performance and functional tests without maintaining servers or extra test infrastructure. I also used the [serverless-artillery](https://github.com/Nordstrom/serverless-artillery) plugin to install the load-testing environment on Lambda, and the [artillery-plugin-datadog](https://www.npmjs.com/package/artillery-plugin-datadog) plugin to send Artillery metrics to Datadog over HTTPS.

<div class="row">
    <div class="col-sm mt-3 mt-md-0">
        {% include figure.liquid path="/assets/img/2021-02-17-a-service-renewal-of-backend-built-upon-aws-lambda-artillery-on-datadog-dashboard.jpg" title="Artillery on Datadog dashboard" class="img-fluid rounded z-depth-1" %}
    </div>
</div>

For rate limiting, I used [nestjs-rate-limiter](https://github.com/ozkanonur/nestjs-rate-limiter), which adds configurable limits.

## Conclusion

I was initially skeptical that we could build a fully functioning system on AWS Lambda. Some problems still aren't solved perfectly — cold starts, for example. Even so, recent AWS updates make it possible to run most of this architecture on Lambda.
