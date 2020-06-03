---
layout: post
title: Consumer-Driven Contract Testing - Part I
status: draft
type: post
published: true
comments: true
date: 2020-06-03 12:00:00
author: mos
header-img: "img/post-bg-01.jpg"
---

This is the first of a series of blog posts about Contract Testing which cover the minimum set of theory and practice necessary for an effective adoption in your team.

### Content:

- Introduction
- Provider Contracts
- Consumer Contracts
- Consumer-Driven Contracts
- Contracts in Microservice Architecture
- Technologies


## Introduction

Contract Testing is a category of testing activity where the data formats and conventions defined by two systems (services) which communicate a business value, is tested against a Mock called "Contract". A service _provides_ a callable API that can be _consumed_ by another (or many) service which create an interaction between parties that needs to be satisfied during the evolution and developing of services which are now coupled. The interaction between services probably rely on a communication layer which can be slow or not reachable which can affect the test results. For this reason, sometimes the best solution is to verify the interaction with a [TestDouble](https://martinfowler.com/bliki/TestDouble.html) which describe the expectations between the parties.


### A Pratical Example

Let's suppose that we have a _Customer_ Service which has an HTTP method that provides customer information in a JSON format upon receiving a customer id:

`HTTP GET: api/v1/customer?id=123`

```JSON
{
  "name": "John",
  "surname": "Doe",
  "social_network": "@johnDoe"
}
```

_Order_ Service which needs customer information to complete an order call _Customer_ Service.

![services](/img/services-01.png "Services")

The actors in this scene are defined in this way:

- _Order_ Service is the _Consumer_ 
- _Customer_ Service is the _Provider_
