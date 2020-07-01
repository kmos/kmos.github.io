---
layout: post
title: Consumer-Driven Contract Testing with Pact and Java - Part II
status: draft
type: post
published: true
comments: true
date: 2020-06-17 12:00:00
author: Giovanni Panice
header-img: "img/pact-bg03.jpg"
---

The [previous post]({% post_url 2020-06-03-contract-tests-part1 %}) explains the principles and motivations behind 
contract testing. Today we give a look to how write consumer-driven contract tests with pact and Java in a SpringBoot 
application.

Pact foundation provides junit5 integration for creation and verification of contracts.

Let's start!

## What you need

The [Java Pact Testing framework](https://github.com/DiUS/pact-jvm) used in this example is the `4.0.0`, 
based on `v3` [specification](https://github.com/pact-foundation/pact-specification/tree/version-3).

Furthermore, you should have:

- jdk 1.8 or later
- maven 3.2+
- a bit of testing knowledge in Spring

All the presented code is on [github](https://github.com/kmos/contract-test-pact-java-example).

---
## The example

![example](/img/contractsp2_01.png "example")

The proposed example is similar to the previous one seen in part I: we have a _provider_ service which expose an
API that given a sentence and a timestamp, it replies an echo response enriched with local timestamp. This API _is consumed_
by another service. To summarize:

_endpoint_ ```POST /api/echo```

_request body_
```JSON
{
    "timestamp": 1593373353,
    "sentence": "hello!"
}
```
_response body_
```JSON
{
    "phrase": "hello! sent at: 1593373353 worked at: 1593373360"
}
```

---
## Consumer side

We are driven by Consumer, so then we start working on consumer side: we add the pact maven dependency in consumer `pom.xml`

```xml
<dependency>
    <groupId>au.com.dius</groupId>
    <artifactId>pact-jvm-consumer-junit5</artifactId>
    <version>4.0.10</version>
</dependency>
```
---
### create a contract

Let's start creating a junit5 test with `PactConsumerTestExt` junit extension:

```java

import au.com.dius.pact.consumer.junit5.PactConsumerTestExt;
import org.junit.jupiter.api.extension.ExtendWith;

@ExtendWith(PactConsumerTestExt.class)
class ConsumerContractTest {
```

in ```@BeforeEach``` method we can assert that the `mockServer` which will serve the contracts is correctly up: 

```java
    @BeforeEach
    public void setUp(MockServer mockServer) {
        assertThat(mockServer, is(notNullValue()));
    }
```

ok, now we can create a contract. A contract can be defined with a method *annotated* with `@Pact` that returns 
a `RequestResponsePact` and provides as parameter `PactDslWithProvider`. All methods annotated with `@Pact` are used to
instrument the _mock server_ through the `PactDslWithProvider` in this way: 

```java
    @Pact(provider = "providerMicroservice", consumer = "consumerMicroservice")
    public RequestResponsePact echoRequest(PactDslWithProvider builder) {

        return builder
                .given("a sentence worked at 1593373360")
                .uponReceiving("an echo request at 1593373353")
                .path(API_ECHO) /* request */
                .method("POST")
                .body(echoRequest) 
                .willRespondWith() /* response */
                .status(200)
                .headers(headers)
                .body(echoResponse)
                .toPact();
    }
```

The Pact DSL provides a fluent API very similar to Spring _mockMvc_: Here we are saying that when the _mock server_ receives
 an _echoRequest_, it should return _200_ and an _echoResponse_. The `given` and the `uponReceiving` method, define 
[the specification in bdd approach](https://martinfowler.com/bliki/GivenWhenThen.html) and the Pact testing framework, 
uses the `given` part to bring the provider into the correct state before executing the interaction defined
in the contract.

---
#### Matchers: build a response with PactDslJsonBody

in the previous step we created an interaction using two json object(_echoRequest_ and _echoResponse_). On the provider side, 
the test verify that the generated response is _perfectly_ equal to the one defined in the contract. 

The Pact testing framework provides also a DSL that permits the definition of different matching case in this way:

```java
    @Pact(provider = "providerMicroservice", consumer = "consumerMicroservice")
    public RequestResponsePact echoRequestWithDsl(PactDslWithProvider builder) {

        PactDslJsonBody responseWrittenWithDsl = new PactDslJsonBody()
                .stringType("phrase", "hello! sent at: X worked at: Y") /* match on type */
                .close()
                .asBody();

        return builder
                .given("WITH DSL: a sentence worked at 1593373360")
                .uponReceiving("an echo request at 1593373353")
                .path(API_ECHO)
                .method("POST")
                .body(echoRequest)
                .willRespondWith()
                .status(200)
                .headers(headers)
                .body(responseWrittenWithDsl)
                .toPact();
    }
```
Here we created a _response_ with `PactDslJsonBody` DSL that defines a match case based on _type_ instead of _value_.
It's possible with `PactDslJsonBody` different match case based on regex or array length.

---
### Verify the contract

Now we can create the real test which verify the contract on the consumer side:

```java
    @Test
    @PactTestFor(pactMethod = "echoRequest")
    @DisplayName("given a sentence with a timestamp, when calling producer microservice, than I receive back an echo sentence with a timestamp")
    void givenASentenceWithATimestampWhenCallingProducerThanReturnAnEchoWithATimestamp(MockServer mockServer) throws IOException {
        
        BasicHttpEntity bodyRequest = new BasicHttpEntity();
        bodyRequest.setContent(IOUtils.toInputStream(echoRequest, Charset.defaultCharset()));

        HttpResponse httpResponse = Request.Post(mockServer.getUrl() + API_ECHO)
                .body(bodyRequest)
                .execute()
                .returnResponse();

        ObjectMapper objectMapper = new ObjectMapper();
        EchoResponse actualResult = objectMapper.readValue(httpResponse.getEntity().getContent(), EchoResponse.class);
        
        assertEquals(expectedResult, actualResult);
    }
```

if we run `mvn test` and we don't have errors, we will see in `./target/pacts` a json file that use the pact formalism
for contracts. We use the generated contract in the provider-side.

---
## Provider side

For the provider, we have a different dependency to add in `pom.xml`:

```xml
		<dependency>
			<groupId>au.com.dius</groupId>
			<artifactId>pact-jvm-provider-junit5</artifactId>
			<version>4.0.10</version>
		</dependency>
```

---
### Verify the contract on provider side

Here is the thing: we need to verify the contract against provider implementation. In the Spring world, it's sounds
like an [integration test which verify the web layer](https://spring.io/guides/gs/testing-web/). So here the magic:

```java
@ExtendWith(SpringExtension.class)
@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.DEFINED_PORT, classes = ProviderApplication.class)
@EnableAutoConfiguration
@AutoConfigureMockMvc
@TestPropertySource(locations = "classpath:application-contract-test.properties")
@Provider("providerMicroservice")
@PactFolder("../consumer/target/pacts")
public class ProviderContractTest {
    
    @Value("${server.host}")
    private String serverHost;
    @Value("${server.port}")
    private int serverPort;

    @BeforeEach
    void setupTestTarget(PactVerificationContext context) {
        context.setTarget(new HttpTestTarget(serverHost, serverPort, "/"));
    }

    @TestTemplate
    @ExtendWith(PactVerificationInvocationContextProvider.class)
    void pactVerificationTestTemplate(PactVerificationContext context) {
        context.verifyInteraction();
    }

    @State("a sentence worked at 1593373360")
    public void sentenceWorkedAt1593373360() {
        when(phraseService.echo(1593373353, "hello!"))
                .thenReturn(new Phrase("hello! sent at: 1593373353 worked at: 1593373360"));
    }

}
```

That's it. As you can see, we have a `@SpringBootTest` with a fixed port and a `@TestPropertySource` that defines
it in order to attach the pact context to the application context with `host` and `port` info.
Obviously there are other ways, like random ports and so on, but the main thing here is to bind both context together.

Another thing here is the `@PactFolder` annotation that points to contracts generated by the consumer. The Pact Framework
search for contracts that belong to the service, and run the verification. 
 
---
#### The `@State` annotation

```java
    @State("a sentence worked at 1593373360")
    public void sentenceWorkedAt1593373360() {
        when(phraseService.echo(1593373353, "hello!"))
                .thenReturn(new Phrase("hello! sent at: 1593373353 worked at: 1593373360"));
    }
```

As previously mentioned, the `given` statement in the consumer contract, define with a business expression, the `state` 
in which the system-under-test, should be during the execution. Following this approach, we define in the provider, a 
method with `@state` annotation, that contains the commands necessary for the correct execution. In our case, we mock 
the business service delegated to execute the _eco logic_. The framework executes the `state` method before calling 
the API defined in the contracts. The real test, in this way, is "reduced" to a simple call: 

```java

    @TestTemplate
    @ExtendWith(PactVerificationInvocationContextProvider.class)
    void pactVerificationTestTemplate(PactVerificationContext context) {
        context.verifyInteraction();
    }

```
---
#### Use a broker

If you have a `broker` that stores the contracts, you can change the `@PactFolder` annotation with `@PactBroker` 
one and define the following plugin in the `pom.xml`:

```xml
    <build>
        <plugins>
            <plugin>
                <groupId>au.com.dius</groupId>
                <artifactId>pact-jvm-provider-maven</artifactId>
                <version>4.0.0</version>
                <configuration>
                    <pactDirectory>target/pacts</pactDirectory>
                    <pactBrokerUrl>${pact.broker.protocol}://${pact.broker.host}</pactBrokerUrl>
                    <projectVersion>${contracts.version}</projectVersion>
                    <trimSnapshot>true</trimSnapshot>
                </configuration>
            </plugin>
        </plugins>
    </build>
```
---
## What's Next

We have covered how to develop and verify a simple contract with java starting from the consumer. We have used the Pact 
DSL and matchers introduced in `v3` spec which is an interesting feature during design & testing. 

In the next posts(I hope), we will see how deploy a broker server to store contracts and how integrate the entire flow with
a CI.