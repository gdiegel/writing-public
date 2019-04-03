# Using neo-problem with a pure Spring Boot service

## Introduction

The excellent [neo-problem][1] library implements the [Problem Details for HTTP APIs][2] standard. Although it works best in web services which use the [JAX-RS programming model for REST endpoints][3], it may also be used with pure Spring.

## Throwing domain specific ProblemExceptions

Neo-problem offers different ways of generating error responses on the server side: Returning a JAX-RS response, using the `net.oneandone.neo.problem.AbstractExceptionToProblemMapper` to map any exception to a problem response and throwing generic `net.oneandone.neo.problem.ProblemException`.
Because we are using spring-web components to configure our service in this example, we can't use JAX-RS `javax.ws.rs.core.Response` objects or the `AbstractExceptionToProblemMapper` provided by neo-problem. Instead, we will use the approach of throwing `ProblemException`s which are specific to our domain.

Independent of the mechanism you probably want to create your own `net.oneandone.neo.problem.ProblemType` enum as outlined in the neo-problem [usage guide][5]:

```java
public enum MyProblemType implements ProblemType {

    //...
    NO_SUCH_PRODUCT(Response.Status.NOT_FOUND, "urn:problem:ui:no-such-product"),
    BAD_CREDENTIALS(Response.Status.UNAUTHORIZED, "urn:problem:ui:bad-credentials");

    private final Response.StatusType status;
    private final URI type;

    MyProblemType(Response.StatusType status, String type) {
        this.status = status;
        this.type = URI.create(type);
    }

    @Override
    public URI getType() {
        return type;
    }

    @Override
    public int getStatus() {
        return status.getStatusCode();
    }
```

These ProblemTypes may then be used when creating an instance of type `net.oneandone.neo.problem.Problem`. [Per default][4], Spring Boot transforms exceptions by way of different implementations of `org.springframework.web.servlet.HandlerExceptionResolver` to error responses like the following:

```
HTTP/1.1 500 Server Error
Connection: close
Content-Type: application/json;charset=UTF-8
Date: Fri, 03 Feb 2017 16:13:30 GMT
Pragma: no-cache
X-Application-Context: my-service:local:8080

{
    "error": "Internal Server Error",
    "exception": "net.oneandone.neo.problem.ProblemException",
    "message": "org.springframework.web.util.NestedServletException: Request processing failed; nested exception is net.oneandone.neo.problem.ProblemException: urn:problem:ui:no-such-product",
    "path": "/products/2",
    "status": 500,
    "timestamp": 1486138410559
}
```

To achieve an rfc7807-compliant response format, we will tell Spring how to handle exceptions of type `ProblemException`.

## Mapping ProblemException to ResponseEntity

We will use the approach of throwing generic `ProblemException` and creating our own exception handler which is able to map these kinds of exceptions to `org.springframework.http.ResponseEntity<T>`. The following snippet shows a way to create such a handler:

```java
@ControllerAdvice
public class MyExceptionHandler {

    @ExceptionHandler(ProblemException.class)
    public ResponseEntity<Problem> handleException(ProblemException e, HttpServletRequest req) {
        final Problem problem = e.getProblem();
        final MediaType mediaType = MediaType.parseMediaType(Problem.JSON_PROBLEM_TYPE.toString());
        return ResponseEntity.status(problem.getStatus()).contentType(mediaType).body(problem);
    }

}
```

Here we are registering a handler for `ProblemException` in our class. The `@org.springframework.web.bind.annotation.ControllerAdvice` annotation is a specialization of `@org.springframework.stereotype.Component` and indicates that the class assists a controller. The singular method is annotated with `@org.springframework.web.bind.annotation.ExceptionHandler(ProblemException.class)`, indicating that it handles all exceptions of this type. We declare the method to return the parametrized type `ResponseEntity<Problem>`, use the actual `Problem` instance to construct the instance of `ResponseEntity<Problem>` in the method body and return this instance. Spring Boot will automatically register the `MyExceptionHandler` class on application startup through classpath scanning. The response for the same request now looks like the following:

```
HTTP/1.1 404 Not Found
Cache-Control: no-cache, no-store, max-age=0, must-revalidate
Content-Type: application/problem+json
Date: Fri, 03 Feb 2017 16:21:50 GMT
Expires: 0
Pragma: no-cache
Transfer-Encoding: chunked
X-Application-Context: my-service:local:8080

{
    "id": "2",
    "status": 404,
    "type": "urn:problem:ui:no-such-product"
}
```

## Handling spring-security exceptions

Exceptions thrown by spring-security are either of type `org.springframework.security.core.AuthenticationException` or `org.springframework.security.access.AccessDeniedException` and are thrown in spring-security's servlet [filter chain][6]. Because of that, they can't be caught with a handler like we've used in the previous paragraph, which applies to a servlet environment.

`org.springframework.security.web.access.ExceptionTranslationFilter` is responsible for handling any AuthenticationException spring-security being thrown. This filter will call an appropriate `org.springframework.security.web.AuthenticationEntryPoint` to handle the failed authentication attempt and present the caller with an appropriate response. Handling `AuthenticationException` may consequently be achieved by configuring our own `AuthenticationEntryPoint` and adding it to the spring-security configuration.

The following snippet show how such an entry point may be implemented:

```java
public class MyAuthenticationEntryPoint implements AuthenticationEntryPoint {

    @Override
    public void commence(HttpServletRequest request, HttpServletResponse response, AuthenticationException exception)
            throws IOException, ServletException {
        final Problem problem = Problem.type(MyProblemType.BAD_CREDENTIALS).withDetail(exception.getMessage()).build();
        response.setHeader(HttpHeaders.WWW_AUTHENTICATE, "Basic realm=\"Realm\"");
        ServerHttpResponse outputMessage = new ServletServerHttpResponse(response);
        outputMessage.setStatusCode(HttpStatus.UNAUTHORIZED);
        new ProblemHttpMessageConverter().write(problem, "application/problem+json", outputMessage);
    }

}
```

This implementation uses a custom `org.springframework.http.converter.HttpMessageConverter<T>` to write to the response object:

```java
final class ProblemHttpMessageConverter extends AbstractHttpMessageConverter<Problem> {

    //...

    @Override
    public boolean supports(Class<?> clazz) {
        return Problem.class.isAssignableFrom(clazz);
    }

    @Override
    public Problem readInternal(Class<? extends Problem> clazz, HttpInputMessage inputMessage) throws IOException {
        return Problem.type(-1, "about:blank")
                .withDetail(StreamUtils.copyToString(inputMessage.getBody(), getDefaultCharset())).build();
    }

    @Override
    public void writeInternal(Problem problem, HttpOutputMessage outputMessage) throws IOException {
        new ObjectMapper().writeValue(outputMessage.getBody(), problem);
    }
}
```

If your application for example allows HTTP Basic authentication, you can to add the filter including the custom entry point to your spring-security configuration like so:

```java
@Configuration
@EnableWebSecurity
static class WebSecurityConfig extends WebSecurityConfigurerAdapter {

    @Override
    protected void configure(HttpSecurity http) throws Exception {
        http.authorizeRequests()
                    .anyRequest().fullyAuthenticated()
                    .and().httpBasic().authenticationEntryPoint(new MyAuthenticationEntryPoint());
    }
}
```

[1]: https://github.com/1and1/neo "neo-problem"
[2]: https://www.rfc-editor.org/rfc/rfc7807.txt "rfc7807"
[3]: http://docs.spring.io/spring-boot/docs/current/reference/html/boot-features-developing-web-applications.html#boot-features-jersey "JAX-RS and Jersey"
[4]: http://docs.spring.io/spring-boot/docs/current/reference/htmlsingle/#boot-features-error-handling "Spring Boot error handling"
[5]: https://git.mamdev.server.lan/mam-java-common/neo/blob/master/neo-problem/README.md "neo-problem Usage"
[6]: http://docs.spring.io/spring-security/site/docs/current/reference/htmlsingle/#security-filter-chain "security-filter-chain"
