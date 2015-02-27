# pact-jvm-provider-spring-mvc

A test runner to help to write provider pact tests with spring mvc, followed [pact spec](https://github.com/bethesque/pact-specification).

There are already several projects help to write provider test in [pact-jvm](https://github.com/DiUS/pact-jvm) repository,
e.g. gradle, maven, sbt, specs2, but we can't find a convenient way to write for spring-mvc projects.

Spring mvc has already provide a test framework to help to write integration test, the API is like:

```java
when(myResponse.getStatusCode()).thenReturn(HttpStatus.OK);
when(myService.signInWithToken(any(String.class))).thenReturn(myResponse);

standaloneSetup(new MyController(myService))
        .build()
        .perform(
            get("/my/users/current")
                .contentType(MediaType.APPLICATION_JSON)
                .header("token", "validToken"))
        .andExpect(status().isOk());
```

You can see it used provided `standaloneSetup` method to wrap a controller to perform request and valid response,
without starting a real server. In this case, we can easily mocking dependencies and do the testing.

Since pact provider test also needs some preparing tasks (mocking), we can follow the same approach to make thing easier
(compared to starting a real server).

How to use it
-------------

This project provides a junit test runner: `PactRunner`, and also some helper annotations.

An example:

```java
import static org.mockito.Matchers.eq;
import static org.mockito.Mockito.when;

import static com.reagroup.pact.provider.PactRunner.uriPathEq;

import java.io.File;
import java.io.IOException;
import java.io.InputStreamReader;

import org.junit.Before;
import org.junit.runner.RunWith;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.beans.factory.annotation.Qualifier;
import org.springframework.core.io.FileSystemResource;
import org.springframework.http.*;
import org.springframework.mock.web.MockHttpServletRequest;
import org.springframework.test.context.ContextConfiguration;
import org.springframework.util.FileCopyUtils;
import org.springframework.web.client.RestTemplate;

import com.reagroup.pact.provider.PactFile;
import com.reagroup.pact.provider.PactRunner;
import com.reagroup.pact.provider.ProviderState;
import controller.MyController;

@RunWith(PactRunner.class)
@ContextConfiguration(locations = {"classpath:applicationContextPactTest.xml"})
@PactFile("file:src/pactProviderTest/resources/consumer-project-provider-project.json")
public class ProviderPactTest {

    @Autowired
    private MyController myController;

    @Autowired
    private RestTemplate restTemplateMock;

    @ProviderState("my-service forbids a request with invalid token")
    public MyController myServiceForbidsARequestWithInvalidToken() throws Exception {
        HttpEntity<String> myRequest = createRequest("request-invalid-token-1.json", "invalid-token");
        when(restTemplateMock.exchange(uriPathEq("/my-service"), eq(HttpMethod.PUT), eq(myRequest), eq(String.class)))
                .thenReturn(new ResponseEntity<>(HttpStatus.FORBIDDEN));
        return myController;
    }

    private HttpEntity<String> createRequest(String file, String token) {
        return new HttpEntity<>(readTrimmedPactFixture(file), createHeadersWithToken(token));
    }

    private HttpHeaders createHeadersWithToken(String token) {
        HttpHeaders headers = new HttpHeaders();
        headers.setContentType(MediaType.APPLICATION_JSON);
        headers.add("token", token);
        return headers;
    }

    private String readTrimmedPactFixture(String fileName) {
        try {
            String path = "src/pactProviderTest/resources/fixtures/" + fileName;
            return FileCopyUtils.copyToString(new InputStreamReader(new FileSystemResource(new File(path)).getInputStream(), "UTF-8")).trim();
        } catch (IOException e) {
            throw new RuntimeException(e);
        }
    }

}
```

### @RunWith(PactRunner.class)

Use `PactRunner` to run this test case, which will do the following:

1. parse the data from the pact json file specified by `@PactFile`, get the all the interactions
2. find all methods annotated with `ProviderState`, which is used for mocking dependencies and finding the controllers to test
3. for each interaction, `PactRunner` will do the preparing based on provider state, `standaloneSetup` the controller, perform
   the request, and compare the response

### @ContextConfiguration(locations = {"classpath:applicationContextPactTest.xml"})

Tells spring where the context file is. Notice we can override some beans in the `...PactTest.xml` by some mocking beans, e.g.
`restTemplateMock`

### @PactFile("file:src/pactProviderTest/resources/consumer-project-provider-project.json")

Tells `PactRunner` where the pact file (upload by consumer) is

### @ProviderState("my-service forbids a request with invalid token")

Mark a method as a preparing method for a specified provider state. The method will do 2 things:

1. Do some mocking in the body
2. Return the controller which will be tested against

The content of `@ProviderState` should be exactly the same as one of the provider state in the pact JSON file, otherwise
an error will be thrown.

### uriPathEq

You can use the `uriPathEq` exposed by object `PactRunner` to check if two URIs path equal to each other without the host part,
which may be useful when writing mocking methods.

Say: `http://aaa.com:8888/hello/world` are equal to `https://bbb.com:9999/hello/world` with `uriPathEq`

Run the test
------------

Just run the test as normal junit test, you will get the reports.
