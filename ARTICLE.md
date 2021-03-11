
WIP: draft 2021-03-11

# How we setup a api mock server for distributed e2e testing

In my current project, we're building a distributed e-commerce solution, consisting of a number of microservices, communicating with supporting, (often) legacy, systems. As in all e-commerce solutions we need to lookup customers, perform credit, risk and fraud checks, payments, lookup offers, products and stock levels, among other things.

Some of these "supporting" systems we use to accomplish all this are already in place. Some are off-the-shelfe products, some are external and some are built by other teams. Either way, we mave more or less control over them, and basically every system has its unique use case on how to setup, and maintain its data.

The idea with standalone mock servers isn't in any way new, and doesn't solve all problems, but we are really satisfied with how easily it was to get started, and the level of confidence we get with relatively low effort. I will deomstrate how we achived this, using a simple wrapper of the excelellent Wiremock framework.

## The problem

As with all distributed platform, there's a challenge to achieve confidence that the platform, as a whole, actually works. At least to some acceptible degree.

I personally believe that such confidence is hard to achieve with unit, component or system testing. With unit or component testing I mean the kind of tests that test a small part of the platform in isolation. Of course these tests have great value in verififying that a specific class, unit, component or system, works as expected. But it only tells ut so much on how a complex distributed platform works as a whole. In theory the whole will work if all the parts are verified. I do think that it can be true, but in my opinion, such test setup can be complex and expensive to maintain.

Of course there's a lot of value in class, unit, component or system tests, but I wouldn't feel confident with the platform unless there's also, manual or automated, high level e2e testing.

Another flavour of testing, which I kind of like, is contract testing; where you verify, not only the parts, but their also their interactions. This will definitely tell us more on how the parts will work as a whole. As mentioned, I do like the idea, but in practice I have experienced problems with getting other teams to adopt to such testing. To test the entire platform, basically all teams and participants needs to be onboard the contract testing train, which will probabaly be big politcal and techical challenge.

Ok, so we want to have some e2e testing. What's the problem. Just start write some tests?!? Sounds easy enough. And yes, the actual tests can be quite easy; fill in some input fields, click some buttons and verify that an order, or whatever, eventually is created.

The first issue with e2e testing is that we need everything up and running, in a, preferable, production like setup. Most of the time we do have that. We usually have some kind of staging or dev environment. Here comes the next problem. Our tests will have dependencies, not only to downstream services, but also on test data. Test data can be a nightmare to setup and maintain. Especially in high level testing, when we're dealing with a complex, distributed platform where we don't have control over all parts.

At the same time time, we want to include as much as possible, of those parts we don't control, to increase the level of confidence that it all works together.

Finally, we also have testers. The purpose of the automated e2e tests is to cover regression testing of the main capabilities of the platform. While manual testing can concentrate on edge cases, and acceptance test of new features. This, together with the individual service's test suites, gives us confidence to allow continuous integration and deployment.

A environment can be complex to setup and maintain and we wanted to avoid setting up a dedicated environment for our automated e2e testing, since that means additional setup and maintenance. The goal was to (re)-use one of our existing environments, and simultaneous support both mocked automated testing, and manual testing, including calls to some, or all legacy system. And that all this should also be simple to setup and maintain.

#To accomplish this we, on merge to master, deploys a new versions of a service. Once the deploy has finisished we run the e2e tests, #and then we may or may not promote it to production.


### Test data

I've seen some a lot of creative solutions. One approach can be to setup test data using the available api's. When possible, this is usually a good approach. We just create some sliced test data and manage to avoid a lot of issues with test data not beeing in place in some data store deep down. Unfortunately this isn't always possible, and then everything blows up once your request reaches some legacy, whatever-system, that recently got a new production data dump. We could of course try to synchronize a set of production data copies, but it will take some work to setup such, and we will be vulerable to changing data.

You could of course use a known data set. Relying on that whatever data our tests needs will be there, and doesn't change. There's defenitely some benefits using that. But we do need to restore it to a known state, to be able to perform regression testing, and we will probably be constrained in the available test cases.

Or maybe, we create our own universal test data builder, that, after some time, starts to be more complex than the actual platform it's supposed to support. It's a sad day when the test data builder gets its own project in jira,

Finally there's allways that issue with that monster those guys'n'girls in the neighbouring team have built, or rather; is trying to build? We do need to use it, but our test crashes and burns each and every day, because they suck!

### Environment


## The solution

As mentioned, our take on this was to setup mock servers, acting as external, or legacy, systems, that we easily can control from the tests.

### The api mock server

The actual api mock server is a simple spring boot application, which can be configured to start a set of WireMockRunners, where each runner acts as a mock for some system.

We start all mocks with wiremock's - proxy-all switch. When running wiremock in this mode, all un-matched requests will be forwarded to the server they're proxying. This gives us the possibility to share environment between automated e2e tests and other. We carefully choose id's that doesn't interfere with existing data.

(picture showing that calls from service A and B always hits the mock, and then is mocked or proxied, depending on any stub is matched)

### Setup mocking

Since the api mock server is based on wiremock we can simply use the well-known wiremock rest api, to interact with the wiremock runners exposed by our api-mock-server. We can register stub mappings, verify behaviour and all the things available when using vanilla wiremock.  We can for example take advantage of wiremocks support for pre-loaded mapping files.

For some systems, (or requests), we have setup static mocks. We pack the api-mock-server with some pre-defined stub files, giving us mocks that allways will be in place. We have though found it very easy-to-use and convenient, to create a number of mock clients. These helps the test writer to setup whatever preconditions the test needs, in each external system.

Example mock client
```
public class CustomerServiceMockClient {

  private WireMock wireMockClient = WireMock.create()
                .https().host("customer-system.com").build());

  public void givenCustomer(Integer ssn, String name) {
        wireMockClient.register(
                get("/api/rest/v1/customers?ssn="+ taxpayerId)
                        .willReturn(aResponse()
                                .withStatus(200)
                                .withHeader("Content-Type", "application/json")
                                .withBody(customer(ssn, name))));
    }

    private String customer(Integer ssn, name) {
      // build and return the doc this system uses to represent a customer 
    }
}
```

Once a set of these mock clients has been created the actual testing could look something like this. An important thing to have in mind, is that, dependeing on, what we select to mock, we are effectively creating a slice of test data, that may or may not, exist in the systems we currently aren't mocking. As I said before, we do need to make a * where we use as many "real" api's as possible, and only what what is necessary to make the test possible to repeat over time.


```
    @BeforeEach
    public void setupTestData() {
        //given
        customerServiceMockClient.givenCustomer("111111", "Test Customer");
        creditCheckServiceMockClient.givenCreditLimit("111111", 5000);
        whateverServiceMockClient.givenSomething(id, testData1, testData2);        
        stockServiceMockClient.givenStockLevel("P111", 5);
    }

    @Test
    public void test() throws Exception {
      //kick off the actual test
    }
    
    @AfterEach
    public void setupTestData() {
        // we either want to reset mocks, to not interfer with other testing,
        // or keep the setup to aid troubleshooting
    }
```

### Recording mode

The api mock server can also be run in recording mode, capturing requests and responses, to help getting started with setting upp mocking. We can start the server in recording mode, and fire off some requests. These requests will be captured into files, which we then can use to create on-demand mocking clients, or static stub files.

## Gotchas

Keeping mocks up to date

Mocks bugs

Mock performance vs. mocked

Troubleshooting failed e2e tests
(tracing and logging)

ResetAll

## Resources

[Wiremock](http://wiremock.org/)
