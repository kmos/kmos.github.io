---
layout: post
title: Consumer-Driven Contract Testing - Part I
type: post
published: true
comments: true
date: 2020-06-03 12:00:00
author: nmosf
header-img: "img/pact-bg02.jpg"
description: "Contract Testing is a category of testing activity where the data formats and conventions defined by two services which communicate a business value, is tested against a Mock called Contract. In this article we will see principles and motivations for adoption of Consumer-Driven Contract Testing"
---

## Introduction

This is the first of a series of blog posts about Contract Testing which cover the minimum set of theory and practice 
necessary for an effective adoption in your team, from design to code integration.

Contract Testing is a category of testing activity where the data formats and conventions defined by two systems 
(services) which communicate a business value, is tested against a Mock called "Contract". A service _provides_ 
a callable API that can be _consumed_ by another (or many) service which create an interaction between parties 
that needs to be satisfied during the evolution and developing of services which are now coupled. The interaction 
between services probably rely on a communication layer which can be slow or not reachable which can affect the test results. 
For this reason, sometimes the best solution is to verify the interaction with a [TestDouble](https://martinfowler.com/bliki/TestDouble.html) 
which describe the expectations between the parties.

For the sake of clarity, I have [strong opinion](https://www.youtube.com/watch?v=oxbS9Pe2PhE&feature=youtu.be).

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
of_ **breaking change** in which _Consumer_ doesn't respect the defined interaction between _Provider_. 

Therefore, the interaction between services can be broken in many ways, and the cause can be triggered by both 
sides which makes necessary some checks to avoid regression during the development.

## e2e Testing

The _simplest_ thing to do, is to create an [end-to-end test](https://martinfowler.com/articles/practical-test-pyramid.html#End-to-endTests) 
that cover the entire calls flow between _Consumer Service_ and _Provider Service_.

![e2e](/img/services-02.png "e2e")


Despite what I previously said about _simplicity_, e2e testing give the best confidence on software behaviour but with some problems.
They are flaky tests and often fail with false positive furthermore e2e tests are hard do maintain, slow, and it's necessary to
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

~~Developers~~ (I) usually dislike maintaining things which depend on environments or that needs hours to have a result.
If something is faulty and slow, it's often untrusted and consequently useless. Furthermore, works with other services 
means deal with other teams which are often too much busy doing new ~~bugs~~ features. So then, to avoid this 
annoying stuff, it's necessary to adopt a strategy which imply a level of complexity which _usually_ developers are used 
to see: **Mocks**.

Mocks are a type of _TestDouble_ that define a sort of specification 
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

###### ![consumer contracts](/img/services-04_1.png "consumer contracts")

We can say that a provider contract is a **closed** and **complete** expression of business functionalities. 
Instead, consumer contract is an **open** and **incomplete** definition of business expectations. When a provider
accepts the expectations defined by the consumer contract, confirms that the expectation is a functionality that supported 
for a period of time.

## Consumer-driven Contracts

![google search true story](/img/googlesearch.jpg "google search true story")

Ok, so now? Well, If you take a look on internet, [~~you will not find~~](https://pactflow.io/blog/the-curious-case-for-the-provider-driven-contract) 
any trace of provider driven contract testing [~~or question about it~~](https://github.com/DiUS/pact-jvm/issues/327).
On the contrary, [the web is full of posts about Consumer-driven contract testing](https://lmgtfy.com/?q=consumer+driven+contract+testing&s=d).
This should be enough for you and me to choose a consumer-driven solution but, as I said at the beginning,
the idea of this post is to give the minimum set of knowledge about Contract testing, and to have an answer
in case of your colleagues are free mind enough to ask if exists an alternative meanwhile you are exposing why Contract
tests are so [cool](https://reflectoring.io/7-reasons-for-consumer-driven-contracts/).

In short terms, it's all about business. Consumer contracts point the finger on current supported business value exposed 
by the provider. In a consumer-driven approach, the sum of all contracts generated by the consumers
and asserted by the provider is **closed** and **complete** respect the functionalities requested.

Anyway Consumer-driven Contracts drive the evolution of the services keeping the focus on what really matter with 
a feedback loop on the changes that will arrive during the life and death of the services which cannot be achievable 
with a Provider-driven approach which not take in account the feedback from the consumer.

In my experience, it's easier and natural to think in a Provider-First manner than Consumer-first, and the reason behind
this mindset, it might be connected to the fact that a breaking change made by the Provider is more detectable than the one
made by the Consumer. In a real world example where your team isn't the owner of both services, you have to deal with meetings, 
longer meetings and extravagant, informal, hermetic, long-winded design docs and other [mythical beasts](https://en.wikipedia.org/wiki/The_Mythical_Man-Month). 
A contract is a formal expression of needs and duties which can be another tool in your pocket that can be used in order
to reduce the background noise during the process of evolution of services. Probably you know the sensation of powerlessness
when a design doc or [swagger definition](https://www.openapis.org/) reaches the mailbox or a shared folder and it's
necessary another meeting or mail tread to have a change in the definition or worse, get an HTTP response status error because
someone made a little change in the API which isn't versioned.

![Consumer Driven Process](/img/services-07.png "Consumer Driven Process")

To summarize, Consumer-Driven contracts can give you a process that presents an iterative way of proceeding with a 
formalised format which can help large organization with services which are owned by different teams that can be in different
locations. 

# Contract Testing with Pact

[Pact.io](https://docs.pact.io/) is an implementation of Consumer-driven contract testing which actually support [many 
languages](https://docs.pact.io/implementation_guides/other_languages). At the moment of writing this post, 
pact foundation released [v3 specification](https://github.com/pact-foundation/pact-specification) which cover the following
features:

- pact format for message queues
- regular expression and type matching
- specification shared between Ruby, JVM and .Net versions

![Pact ecosystem](/img/services-05_1.png "Pact ecosystem")

Furthermore, it's available a [broker](https://github.com/pact-foundation/pact_broker) which can be used to publish 
and share contracts between services.


## What's Next

We have covered the minimum set of principles and motivations that are necessary in my opinion to work with
contract testing. In the next posts we will take a look to how:

- create consumer-driven contracts tests with Pact and Java
- setup broker server
- integration with Continuous Integration

## References
> - [Consumer-Driven Contracts: A Service Evolution Pattern](https://martinfowler.com/articles/consumerDrivenContracts.html#Schematron)
> - [ContractTest](https://martinfowler.com/bliki/ContractTest.html)
> - [TestDouble](https://martinfowler.com/bliki/TestDouble.html)
> - [The curious case for the Provider Driven Contract](https://pactflow.io/blog/the-curious-case-for-the-provider-driven-contract)
> - [The Practical Test Pyramid](https://martinfowler.com/articles/practical-test-pyramid.html)
> - [Patterns for Managing Source COde Branches](https://martinfowler.com/articles/branching-patterns.html)
> - [Mocks Aren't Stubs](https://martinfowler.com/articles/mocksArentStubs.html)
> - [7 Reasons to Choose Consumer-Driven Contract Tests Over End-to-End Tests](https://reflectoring.io/7-reasons-for-consumer-driven-contracts/)