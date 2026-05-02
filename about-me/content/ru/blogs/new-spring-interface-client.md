+++
title = "HttpClient в Spring 7: замена FeignClient или нет?"
date = 2026-04-13T18:14:00+03:00
draft = false
description = "Как отказаться от FeignClient и перейти на новый Spring HttpClient"
tags = ["spring", "HttpExchange", "HttpClient", "openapi", "java"]
categories = ["backend", "spring"]
+++

Статья есть на [Habr'е](https://habr.com/ru/articles/1022466/)

За последние несколько лет для вызова внешних API в каждом втором (если не первом) проекте я видел одну и ту же картину:
- `RestTemplate`
- или `FeignClient`

Причём Feign почти всегда шёл в связке с OpenAPI: сгенерировали клиент, получили интерфейсы и не думали о реализации. Удобно, красиво, привычно.

Но потом в Spring появился нативный декларативный HttpClient, который работает поверх `RestClient` / `WebClient

И у меня возник вопрос:  **а можно ли им заменить Feign, не потеряв удобство?**

Спойлер: да, можно.

## Откуда вообще взялся HttpClient

Идея, на самом деле, очень простая.

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

В Spring уже есть:

- `@RestController` – принимаем HTTP запросы
- аннотации вроде `@GetMapping`, `@RequestParam`, `@PathVariable`

Так почему бы не использовать **тот же подход**, но для исходящих запросов?
Так появился `HttpClient`. Те же аннотации, тот же стиль – только теперь это клиент.

Если вы раньше работали с Feign – здесь может возникнуть логичный вопрос: разве это не то же самое? По ощущениям – очень похоже:
- интерфейс
- аннотации
- декларативный вызов

Но есть важное отличие:
Feign – это часть Spring Cloud и отдельная экосистема со своей инфраструктурой.  
А `HttpExchange` – это нативный механизм Spring Framework, который работает поверх стандартного HTTP-клиента (`RestClient` или `WebClient`) и не требует дополнительного стека. Плюс более лёгкая настройка и интеграция с Observability из коробки.

То есть внешне они выглядят почти одинаково, но `HttpExchange` – это “тот же подход”, только встроенный прямо в Spring.

### Немного эволюции

В Spring 6 это всё уже было, но ощущалось немного «сыро».

Потом постепенно:
- появился `RestClient` (`RestTemplate` начали отправлять на пенсию)
- добавили observability, tracing и т.д.

И вот в Spring 7 это уже полноценный инструмент, на который явно делают ставку.

### А что с Feign?

Начиная с 2022 года, `FeignClient` [официально перешёл в стадию поддержки](https://spring.io/blog/2022/12/16/spring-cloud-2022-0-0-codename-kilburn-has-been-released#spring-cloud-openfeign-feature-complete-announcement).
То есть он никуда не исчез, но активного развития уже нет.

И логично, что я начал смотреть в сторону нового `HttpClient` как замены.

## Что насчёт интеграции с openapi-generator

Feign хорош не только декларативностью.

Один из его главных плюсов – работа в паре с **openapi-generator**:
- берём `openapi.yaml`
- прогоняем через `openapi-generator`
- получаем готовый клиент

И просто используем готовый бин:
```java
@Component
@RequiredArgsConstructor
public WeatherClient {
	// Декларативный сгенерированный интерфейс от feign
	private final WeatherApi api;

	public Forecast getWeather(...) {
		var weather = api.getWeather(...);
		// ...
	}
}
```

И для HttpExchange эта опция тоже доступна.
### Как это настроить

Ключевая вещь – это генератор:

```xml
<library>spring-http-interface</library>
```

Полная конфигурация:

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

На выходе получаем:
```java
public interface OpenMeteoApi {

    @HttpExchange(method = "GET", value = "/v1/forecast")
    ResponseEntity<ForecastResponse> getForecast(
        @RequestParam("latitude") Double latitude,
        @RequestParam("longitude") Double longitude
    );
}
```
То есть это **тот же подход, но на нативном Spring API**.
#### Как это подключается

```java
// Такой подход через имплементацию сразу добавляет Observability внутрь клиента
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

#### Настройка обработки ошибок
```java
builder.baseUrl(properties.baseUrl())  
       .defaultStatusHandler(  
               status -> status.is4xxClientError() || status.is5xxServerError(),
               //errorHandler реализует RestClient.ResponseSpec.ErrorHandler
               errorHandler  
       );
```

```java
public class HttpErrorHandler implements RestClient.ResponseSpec.ErrorHandler

//...

@Override  
public void handle(HttpRequest request, ClientHttpResponse response)
```
То есть мы можем обрабатывать 4хх и 5хх статусы сразу на месте, по любой логике, которая нам нужна. Либо можем обернуть это в нашу ошибку, например, `ApiRequestException`, и затем настроить `@ControllerAdvice` – всё зависит от вашего воображения.

Также в новом клиенте легче настраивается работа с Resilience4j, интерцепторами.

## А как же WebClient?

Да, можно использовать и его.

Можно даже генерировать реактивные сигнатуры, нужно всего лишь ~~перейти по ссылке для увеличения...~~ добавить в генератор соответствующий флажок и настроить в конфиге не RestClient builder, а WebClient.Builder

```java
Mono<ForecastResponse> // или Flux
```

Но тут появляется интересный момент - **Project Loom** и его работа из коробки в Spring.
```properties
spring.threads.virtual.enabled=true
```

И внезапно:
- блокирующий код перестаёт быть проблемой
- `RestClient` становится «достаточно хорошим»
- код остаётся простым, те пишем код как раньше в блокирующем стиле, никакой реактивщины

Остаётся дождаться Structured Concurrency и тогда конкурентный код с помощью VT станет намного user-friendly.

Лично моё мнение:  
`WebClient` приносит с собой реактивный стиль, который не всегда просто читать и дебажить.  
В сценариях без реактивных пайплайнов `RestClient + virtual threads` даёт схожую масштабируемость для I/O, но с более понятным императивным кодом.

## Самая неожиданная проблема - валидация

Вот тут я залип.

Сгенерированные Response выглядят примерно так:

```java
public class ForecastResponse {
	// ...
    private CurrentWeather currentWeather; // без @Valid
}
```

И тут важный момент: Bean Validation не идёт вглубь, так как генератор не ставит аннотации @Valid на вложенных сущностях.
То есть валидируется только верхний уровень.

### Что с этим делать?

Я нашёл три варианта:
#### 1. Аннотации через OpenAPI

```yaml
x-field-extra-annotation: "@jakarta.validation.Valid"
```

Работает, но требует менять спецификацию. А хотелось бы бездумно копировать спеку.
#### 2. TraversableResolver

Можно заставить валидатор всегда обходить всё дерево.
```java
public class AlwaysTraversableResolver implements TraversableResolver
```
Но:
- риск циклов
- возможный удар по производительности
- влияет глобально

Звучит как «можно, но лучше не надо».
#### 3. Кастомные шаблоны генерации

Самый адекватный вариант:
- переопределяем mustache-шаблоны
- добавляем `@Valid` автоматически

### Тут меня понесло дальше

> Как же правильно валидировать контракт быстро? И как сделать так, чтобы мы сразу падали, если внешний API промазал мимо контракта? Те есть в поле notNull пришло null.

В такие моменты кажется, что лучший способ валидировать REST контракт – это не использовать REST и перейти на gRPC 🤓  
Но gRPC не панацея.

Всё же, хочется придумать решение для этой проблемы.

Я подумал:
> а что, если базовую валидацию делать во время SerDe?

И тут появляется Kotlin.
```kotlin
data class ForecastResponse(
    val temperature: Double,
    val windSpeed: Double?
)
```
Идея такая: чтобы вынести логику генерации `HttpClient` в отдельный Spring starter. То есть этот репозиторий будет хранить `openapi` спеки, и `openapi-generator` будет генерировать Kotlin классы.

Kotlin тут нужен как первая станция защиты – проверка на null safety.
По идее, конечный докер образ не должен сильно распухнуть, так как нужно в стартер добавить две зависимости (помимо HttpClient и openapi-generator):
- `org.jetbrains.kotlin:kotlin-stdlib`
- `com.fasterxml.jackson.module:jackson-module-kotlin`
  Много они не весят – примерно 5MB.

Если API внезапно вернёт `null` в non-null поле – мы упадём сразу на десериализации.  
И это удобно:
- ошибка максимально ранняя
- никаких неожиданных NPE дальше

При этом важно понимать, что такое поведение зависит от конфигурации Jackson и kotlin-module, а также от того, как именно сгенерированы модели.  
То есть это не абсолютная гарантия, а скорее способ добиться fail-fast поведения для части ошибок контракта, в первую очередь связанных с nullability.

А `@Min`, `@Max` и прочее можно оставить как второй слой и проверять конкретно в каждом сервисе, который будет использовать этот стартер.
___
## Финал

Если вы уже переходите на 6 или 7 Spring, можно спокойно использовать новый декларативный HttpClient.

Если у вас пару вызовов внешнего API – проще будет вручную написать новый декларативный интерфейс.
Если же внешних API вызовов много, или у вашего сервиса много RestController'ов, и вы хотите поставить client коллегам внутри Java Spring микросервисов - это хорошая возможность воспользоваться `openapi-generator` в связке с новым HttpClient.

Подводя итог, стек получается такой:
- **HttpСlient** → декларативный клиент + возможность работы в синхронном стиле через `RestClient` либо же реактивно через ProjectReactor - `WebClient`
- **OpenAPI** → генерация интерфейсов и классов
- **Bean Validation** → второй этап валидации на стороне пользователя стартера

и возможно рассмотреть вариант генерации Kotlin как защита от null (fail fast).

---
👉 Репозиторий с примером:  
https://github.com/Ulllie/http-exchange-openapi-gen
