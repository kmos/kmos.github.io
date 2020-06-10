---
layout: post
title: Consumer-Driven Contract Testing - Part I
status: draft
type: post
published: true
comments: true
date: 2020-06-03 12:00:00
author: Giovanni Panice
header-img: "img/pact-bg02.jpg"
---

This is the first of a series of blog posts about Contract Testing which cover the minimum set of theory and practice 
necessary for an effective adoption in your team, from design to code integration.

## Content:

- [Introduction](#introduction)
- [Context](#context)
- [e2e testing](#e2e-testing)
- [Mocks](#mocks)
- [Consumer-Driven Contracts Testing](#consumer-driven-contracts)
- [Contract Testing with Pact](#contract-testing-with-pact)
- [References](#references)

## Introduction

Contract Testing is a category of testing activity where the data formats and conventions defined by two systems 
(services) which communicate a business value, is tested against a Mock called "Contract". A service _provides_ 
a callable API that can be _consumed_ by another (or many) service which create an interaction between parties 
that needs to be satisfied during the evolution and developing of services which are now coupled. The interaction 
between services probably rely on a communication layer which can be slow or not reachable which can affect the test results. 
For this reason, sometimes the best solution is to verify the interaction with a [TestDouble](https://martinfowler.com/bliki/TestDouble.html) 
which describe the expectations between the parties.


## Context

Let's suppose that we have a _Product_ Service which has an HTTP method that provides product information in a JSON 
format upon receiving a product id:

`HTTP GET: api/v1/product?id=123`

```JSON
{
  "name": "nexus",
  "type": "smartphone",
  "price": "21.03"
}
```

The product information are consumed by _Customer_ Service.

![producer-consumer](/img/services-01.png "producer-consumer")


Conventions are defined in this way:

- _Customer_ Service is the _Consumer_ 
- _Product_ Service is the _Provider_


During developing, the _Product_ service can evolve the API in a way that the HTTP response change the data format. 
For example, the _type_ field from a string value change to object:

```JSON
{
  "name": "nexus",
  "type": {
    "category": "smartphone",
    "weight": 0.2,
    "color": "blue"
  },
  "price": "21.03"
}
```

this is defined as a **breaking change** in which _Provider_ doesn't respect the defined interaction between _Consumer_.

In the same way, the Customer service can evolve the request call. For example, instead of call the API with a query 
parameter ```id```, it can use a field in the header to query the Product Service. Also, this example is a _sort 
of_ **breaking change** in which _Consumer_ doesn't respect the defined interaction between _Provider_. These examples 
with the inherent differences, help to define some principles and characteristics of services that we'll cover later.

## e2e Testing

As previously seen, the interaction between services can be broken in many ways, and the cause can be triggered by both 
sides which makes necessary some checks to avoid regression during the development.
The _simplest_ thing to do, is to create an [end-to-end test](https://martinfowler.com/articles/practical-test-pyramid.html#End-to-endTests) 
that cover the entire calls flow between _Consumer Service_ and _Provider Service_.

![e2e](/img/services-02.png "e2e")


Despite what I previously said about _simplicity_, e2e testing give the best confidence on software behaviour but with some problems.
They are flaky and often fail with false positive furthermore tests are hard do maintain, slow, and it's necessary to
span multiple services in testing environment with terrible slowness caused by deploy flow and environment nightmares.
There are some solutions ([1](https://www.soapui.org/), [2](https://github.com/postmanlabs/newman)) that can provide the
right balance between _simplicity_ and _maintainability_ which are a step back e2e tests but often you need a staging 
environment to execute tests verification or add a level of complexity to remove the environment need, which 
personally I found with these solutions a tedious process. Moreover, all these solutions are _reactive_ approaches which
in the simplest case, doesn't avoid the integration of code that breaks the services interactions, but they provide a sort 
of alerting like open a Jira defect or github issue. It's possible to avoid code integration in case of breaks but 
imply an overcomplicated CI or some [experimental branch](https://martinfowler.com/articles/branching-patterns.html#experimental-branch). 

## Mocks

![Fake news](https://media.giphy.com/media/26n6ziTEeDDbowBkQ/giphy.gif)

~~Developers~~ (I) usually dislike maintaining things dependant of environments or that needs hours to have a result.
If something is faulty and slow, it's often untrusted and consequently useless. Furthermore, works with other services 
means deal with other teams which are often too much busy to insert new ~~bugs~~ feature. So then, to avoid this 
annoying stuff, it's necessary to adopt a strategy which imply a level of complexity which _usually_ developers are used 
to see: **Mocks**.

Mocks are a type of [TestDouble](https://martinfowler.com/bliki/TestDouble.html) that define a sort of specification 
based on expectations and, in this particular case, mocks can substitute APIs or clients reducing in this way, parties
to set up and run.

![services with mocks](/img/services-03.png "services with mocks")

Even if the image show _deployed_ services, there are many [solutions](https://www.baeldung.com/spring-boot-testing#integration-testing-with-springboottest) 
to load part of an application which doesn't imply to run effectively the service. however, you can simulate an interaction
through a [mock or stub](https://martinfowler.com/articles/mocksArentStubs.html) based on **Contracts**. As showed before,
we can have two categories of breaking change based on the side that doesn't maintain expectations and in the same way,
we can identify two categories of contracts:

- Provider Contracts 
- Consumer Contracts

### Contracts characteristics

A _Provider_ service exposes a set of business functionalities which can be **used or not** by one or many consumers 
and ingested in different ways. A change in the provider interface can break interactions with many consumers, but it is
not the same for the consumer. In the same way, a Provider contract cover completely the functionalities exposed by 
the service with only one definition. On the contrary, a Consumer contract can cover only a set of functionalities 
that are interesting for the service. 

We can say that a provider contract is a **closed** and **complete** expression of business functionalities. 
Instead, consumer contract is an **open** and **incomplete** definition of business expectations. When a provider
accepts the expectations defined by the consumer contract, confirms that the expectation is a functionality that supported 
for a period of time.

![consumer contracts](/img/services-04.png "consumer contracts")

## Consumer-driven Contracts


![Consumer Driven Process](/img/services-06.png "Consumer Driven Process")



# Contract Testing with Pact

![Pact ecosystem](/img/services-05.png "Pact ecosystem")

## Next

## References