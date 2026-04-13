+++
title = "HttpClient in Spring 7: a replacement for FeignClient or not?"
date = 2026-04-04T18:14:00+03:00
draft = false
description = "How to move away from FeignClient and adopt the new Spring HttpClient approach"
tags = ["spring", "HttpExchange", "HttpClient", "openapi", "java"]
categories = ["backend", "spring"]
+++

Over the last few years, in almost every other project I worked on, I kept seeing the same picture:

- `RestTemplate`
- or `FeignClient`

And Feign was almost always paired with OpenAPI: generate the client, get the interfaces, and do not think about the implementation. Convenient, clean, familiar.

But then Spring introduced a native declarative HTTP client built on top of `RestClient` / `WebClient`.

And that made me ask a simple question:

**can it replace Feign without losing convenience?**

Spoiler: yes, it can — and in some cases, it probably **should**.

## Where HttpClient came from

The idea is actually very simple.

```java
public interface UserClient {

    @GetExchange("/api/users/{id}")
    UserResponse getUser(
        @PathVariable("id") Long id,
        @RequestParam(name = "includeDetails", defaultValue = "false") boolean includeDetails,
        @RequestHeader("Authorization") String authToken,
        @RequestHeader("X-Request-Id") String requestId
    );

    @PostExchange("/api/users")
    UserResponse createUser(
        @RequestBody @Valid CreateUserRequest request,
        @RequestHeader("Authorization") String authToken
    );

    @PatchExchange("/api/users/{id}")
    void updateUserEmail(
        @PathVariable("id") Long id,
        @RequestParam("email") String email,
        @RequestHeader("Authorization") String authToken
    );
}
```

Spring already has:

- `@RestController` for handling incoming HTTP requests
- annotations like `@GetMapping`, `@RequestParam`, `@PathVariable`

So why not use **the same approach** for outgoing requests?

That is how `HttpClient` appeared. Same annotations, same style — but now it is used on the client side.

If you have worked with Feign before, the obvious question is: is this basically the same thing? In terms of developer experience, it feels very similar:

- interface
- annotations
- declarative calls

But there is one important difference.

Feign is part of Spring Cloud and comes with its own ecosystem and infrastructure.  
`HttpExchange`, on the other hand, is a native Spring Framework mechanism built directly on top of the standard HTTP client stack (`RestClient` or `WebClient`) and does not require an extra layer. It is lighter to configure and integrates more naturally with Spring Observability out of the box.

So yes, from the outside they look almost the same — but `HttpExchange` is essentially the same idea built directly into Spring itself.

### A bit of evolution

Spring 6 already had all of this, but it still felt a little rough around the edges.

Then, gradually:

- `RestClient` appeared (`RestTemplate` started moving toward retirement)
- observability, tracing, and related support became much better

And by Spring 7, this looks like a fully mature tool that Spring is clearly investing in.

### What about Feign?

Starting in 2022, `FeignClient` [officially moved into maintenance mode](https://spring.io/blog/2022/12/16/spring-cloud-2022-0-0-codename-kilburn-has-been-released#spring-cloud-openfeign-feature-complete-announcement).

That does not mean it is gone. It is still there. But active feature development is no longer the focus.

So naturally, I started looking at the new `HttpClient` as a replacement.

## What about openapi-generator integration?

Feign is not great only because it is declarative.

One of its biggest strengths is how well it works together with **openapi-generator**:

- take `openapi.yaml`
- run it through `openapi-generator`
- get a ready-to-use client

Then you simply use the generated bean:

```java
@Component
@RequiredArgsConstructor
public WeatherClient {
    // declarative generated Feign interface
    private final WeatherApi api;

    public Forecast getWeather(...) {
        var weather = api.getWeather(...);
        // ...
    }
}
```

And this option is also available for `HttpExchange`.

### How to configure it

The key part is the generator library:

```xml
<library>spring-http-interface</library>
```

Full configuration:

```xml
<plugin>  
    <groupId>org.openapitools</groupId>  
    <artifactId>openapi-generator-maven-plugin</artifactId>  
    <version>${openapi-generator.version}</version>  
    <executions>  
       <execution>  
          <id>generate-weather-client</id>  
          <goals>  
             <goal>generate</goal>  
          </goals>  
          <configuration>  
             <inputSpec>${project.basedir}/src/main/resources/openapi/external/weather-api.yaml</inputSpec>  
             <generatorName>spring</generatorName>  
             <library>spring-http-interface</library>  
             <output>${project.build.directory}/generated-sources/openapi-client</output>  
             <apiPackage>ulllie.exchange.openapi.gen.client.api</apiPackage>  
             <modelPackage>ulllie.exchange.openapi.gen.client.model</modelPackage>  
             <generateApis>true</generateApis>  
             <generateModels>true</generateModels>  
             <generateSupportingFiles>false</generateSupportingFiles>  
             <configOptions>  
                <useSpringBoot4>true</useSpringBoot4>  
                <useJackson3>true</useJackson3>  
                <interfaceOnly>true</interfaceOnly>  
                <skipDefaultInterface>true</skipDefaultInterface>  
                <useBeanValidation>true</useBeanValidation>  
                <annotationLibrary>none</annotationLibrary>  
                <serializationLibrary>jackson</serializationLibrary>  
                <useTags>true</useTags>  
             </configOptions>  
          </configuration>  
       </execution>
    </executions>
</plugin>
```

The output looks like this:

```java
public interface OpenMeteoApi {

    @HttpExchange(method = "GET", value = "/v1/forecast")
    ResponseEntity<ForecastResponse> getForecast(
        @RequestParam("latitude") Double latitude,
        @RequestParam("longitude") Double longitude
    );
}
```

So this is essentially **the same idea, but based on native Spring APIs**.

#### How to wire it

```java
// This implementation-based approach also brings Observability into the client out of the box
@Configuration  
@ImportHttpServices(group = "weather", types = OpenMeteoApi.class)  
public class WeatherRestClientConfig implements RestClientHttpServiceGroupConfigurer {

    @Override  
    public void configureGroups(Groups<RestClient.Builder> groups) {  
        groups.filterByName("weather").forEachClient(
            ($, builder) -> builder.baseUrl("https://api.weather.com")
        );  
    }  
}
```

#### Error handling setup

```java
builder.baseUrl(properties.baseUrl())  
       .defaultStatusHandler(  
               status -> status.is4xxClientError() || status.is5xxServerError(),
               // errorHandler implements RestClient.ResponseSpec.ErrorHandler
               errorHandler  
       );
```

```java
public class HttpErrorHandler implements RestClient.ResponseSpec.ErrorHandler

// ...

@Override  
public void handle(HttpRequest request, ClientHttpResponse response)
```

So we can handle 4xx and 5xx responses right there, with whatever logic we need.

Or we can wrap them into our own exception, for example `ApiRequestException`, and then handle everything centrally through `@ControllerAdvice` — it is entirely up to you.

The new client also makes it easier to integrate things like Resilience4j and interceptors.

## What about WebClient?

Yes, you can use that too.

You can even generate reactive signatures — you just need ~~to click here to enlarge...~~ to add the proper generator flag and configure `WebClient.Builder` instead of `RestClient.Builder`.

```java
Mono<ForecastResponse> // or Flux
```

But this is where another interesting piece enters the conversation: **Project Loom** and its out-of-the-box support in Spring.

```properties
spring.threads.virtual.enabled=true
```

And suddenly:

- blocking code stops being a major problem
- `RestClient` becomes “good enough”
- the code stays simple: still imperative, still easy to read, no reactive overhead

Now we are just waiting for Structured Concurrency to make VT-based concurrent code even more user-friendly.

My personal take is this:

`WebClient` brings a reactive programming style that is not always easy to read or debug.  
In scenarios without reactive pipelines, `RestClient + virtual threads` gives similar I/O scalability while keeping the code much easier to reason about.

## The most unexpected problem: validation

This is where I got stuck for a while.

Generated response models usually look something like this:

```java
public class ForecastResponse {
    // ...
    private CurrentWeather currentWeather; // no @Valid here
}
```

And this detail matters.

Bean Validation does not cascade into nested objects automatically if the generator does not put `@Valid` on nested fields.

That means only the top-level object gets validated.

### What can be done about it?

I found three options.

#### 1. Add annotations directly in OpenAPI

```yaml
x-field-extra-annotation: "@jakarta.validation.Valid"
```

This works, but it requires modifying the specification. Ideally, I wanted to keep the spec copy-paste friendly and untouched.

#### 2. TraversableResolver

You can force the validator to walk the full object graph.

```java
public class AlwaysTraversableResolver implements TraversableResolver
```

But:

- there is a risk of cycles
- it may have a performance cost
- it affects validation globally

So this feels like a “possible, but probably not a great idea” solution.

#### 3. Custom generator templates

This looks like the most practical approach:

- override the mustache templates
- add `@Valid` automatically where needed

### And this is where I went one step further

> What is the right way to validate the contract quickly?  
> How do we fail fast if an external API returns something that breaks the contract?

In moments like this, it is tempting to say that the best way to validate a REST contract is to stop using REST and move to gRPC 🤓

But gRPC is not a magic answer either.

Still, I wanted a better solution for this specific problem.

Then I thought:

> what if we do baseline validation during SerDe?

And that is where Kotlin comes in.

```kotlin
data class ForecastResponse(
    val temperature: Double,
    val windSpeed: Double?
)
```

The idea is to move `HttpClient` generation logic into a separate Spring starter.  
That repository would store OpenAPI specs, and `openapi-generator` would generate Kotlin classes from them.

Kotlin here acts as the first line of defense through null-safety.

In theory, the final Docker image should not grow too much, because in addition to `HttpClient` and `openapi-generator`, the starter would only need two more dependencies:

- `org.jetbrains.kotlin:kotlin-stdlib`
- `com.fasterxml.jackson.module:jackson-module-kotlin`

They are not very large — roughly around 5 MB together.

If the API suddenly returns `null` for a non-null field, deserialization should fail immediately.

And that is useful because:

- the failure happens as early as possible
- there are no strange NPEs later in the flow

That said, it is important to understand that this behavior still depends on Jackson configuration, `jackson-module-kotlin`, and on how the models were generated. So this is not an absolute guarantee — it is more a way to achieve fail-fast behavior for a subset of contract violations, mainly around nullability.

Then things like `@Min`, `@Max`, and similar constraints can remain as a second validation layer on the consumer side of the starter.

## Final thoughts

If you are already moving to Spring 6 or 7, the new declarative `HttpClient` is absolutely something you can use with confidence.

If your service only makes a couple of external API calls, it is probably simpler to write a small declarative interface by hand.

But if you have many external integrations, or your service exposes many `RestController`s, or you want to ship reusable clients to other Java/Spring microservices inside your company, then this is a very good opportunity to use `openapi-generator` together with the new `HttpClient`.

So the resulting stack looks like this:

- **HttpClient** -> declarative client with a choice between synchronous style through `RestClient` or reactive style through Project Reactor with `WebClient`
- **OpenAPI** -> generation of interfaces and models
- **Bean Validation** -> second-stage validation on the consumer side of the starter

And possibly:

- **Kotlin models** -> null-safety as a fail-fast guardrail
---

👉 Example repository:  
https://github.com/Ulllie/http-exchange-openapi-gen
