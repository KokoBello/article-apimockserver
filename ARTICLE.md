# How we setup a api mock server for distributed e2e testing

In my current project, we're building a distributed e-commerce solution, consisting of a number of microservices, communicating with supporting, (often) legacy, systems. As in all e-commerce solutions we need to lookup customers, perform credit, risk and fraud checks, payments, lookup offers, products and stock levels, among other things.

Some of these "supporting" systems we use to accomplish all this are already in place. Some are off-the-shelfe products, some are external and some are built by other teams. Either way, we mave more or less control over them, and basically every system has its unique use case on how to setup, and maintain its data.

## The problem

To be able to quickly deploy features, hotfixes, configurations or whatever. We need confidence that the platform as a whole actually works, at least to some acceptible degree. To accomplish this we, on merge to master, deploys a new versions of a service. Once the deploy has finisished we run the e2e tests, and then we may or may not promote it to production.

As often with high level testing; test data can be a nightmare to setup and maintain. Especially when we're dealing with a complex, distributed platform where we don't have control over some parts.

At the same time time, we want to include as much as possible, of those parts we don't control, to increase the level of confidence it all works together.

Finally, we also have testers. The purpose of the automated e2e tests is to cover regression testing of the main capabilities of the platform. While manual testing can concentrate on edge cases, and acceptance test of new features. This, together with the individual service's test suites, gives us confidence to allow continuous integration and deployment.

A environment can be complex to setup and maintain and we wanted to avoid setting up a dedicated environment for our automated e2e testing, since that means additional setup and maintenance. The goal was to (re)-use one of our existing environments, and simultaneous support both mocked automated testing, and manual testing, including calls to some, or all legacy system. And that all this should also be simple to setup and maintain.

The idea with mock servers isn't in any way new, but we are really satisfied with how easy this was accomplished, using a simple wrapper of the excelellent  Wiremock framework.

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
