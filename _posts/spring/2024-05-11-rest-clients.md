# REST Clients

## 목표

- 간단한 REST API 호출을 할 수 있다.
- 장단점을 설명할 수 있다.
- 상황에 따라 어떤 방식을 사용할지 결정할 수 있다.

## RestClient

modern, fluent API 방식을 제공하는 synchronous HTTP client (thread safe)

Spring Framework6.1 버전부터 제공한다.

RestTemplate와 비슷한 기능을 제공하지만 가독성이 좋고 사용하기 편리하다는 장점이 있다.

```java
// GET request
RestClient defaultClient = RestClient.create();

YesNoApiResponse result = defaultClient.get()
            .uri("https://yesno.wtf/api")
            .accept(MediaType.APPLICATION_JSON)
            .retrieve()
            .body(YesNoApiResponse.class);
```

## WebClient

RestTemplate을 대체 하기 위해 Spring 5.0에 업데이트된 HTTP Client

async 방식을 지원하는 HTTP Client

```java
// GET Blocking request
WebClient webClient = WebClient.create();

ResponseEntity<YesNoApiResponse> result = webClient.get()
    .uri("https://yesno.wtf/api")
    .accept(MediaType.APPLICATION_JSON)
    .retrieve()
    .toEntity(YesNoApiResponse.class)
    .block();
```

- Non-Blocking I/O
- Reactive Streams back pressure
- High concurrency with fewer hardware resources
- Functional-style, fluent API that takes advantage of Java 8 lambdas
- Synchronous / Asynchronous interactions
- Streaming up to or streaming down from a server

## RestTemplate

동기 방식의 HTTP Client

Spring 공식 문서에서 RestTemplate을 RestClient로 migration 하는 방법을 소개하고 있고, 각 기능에 해당하는 RestClient 기능을 소개하고 있다.

sync 방식의 HTTP Client가 필요하다면 RestClient를 사용하자.

## HTTP Interface

`@HttpExchange`를 사용하면 HTTP 서비스를 Java Interface로 정의할 수 있다.

HttpServiceProxyFactory에 interface를 전달하여 RestClient나 WebClient와 같은 HTTP Client를 이용해 요청을 수행하는 proxy를 생성할 수 있다.

ErrorHandling의 경우 사용하는 HTTP Client에 설정하면 된다.

개인적으로 HTTP Interface는 REST Client의 기능을 하는 것이 아니고, HTTP 요청을 한단계 감싼 interface를 사용함으로써 이점이 있다고 생각한다.

RestClient, WebClient 등 어떤 RestClient를 사용하든지 상관없기 때문에 더 큰 유연성을 제공한다.

```java
interface YesOrNoService {
    @GetExchange
    YesOrNoResponse yesOrNo();
}
```

```java
RestClient restClient = RestClient.builder().baseUrl("https://yesno.wtf/api").build();
RestClientAdapter adapter = RestClientAdapter.create(restClient);
HttpServiceProxyFactory factory = HttpServiceProxyFactory.builderFor(adapter).build();

YesOrNoService service = factory.createClient(YesOrNoService.class);
```

```java
WebClient webClient = WebClient.builder().baseUrl("https://yesno.wtf/api").build();
WebClientAdapter adapter = WebClientAdapter.create(webClient);
HttpServiceProxyFactory factory = HttpServiceProxyFactory.builderFor(adapter).build();

YesOrNoService service = factory.createClient(YesOrNoService.class);
```

## 정리

- sync 방식을 사용하여 REST API를 이용하는 경우 RestClient를 사용하자.
- RestTemplate보다는 modern, fluent API를 지원하는 RestClient를 사용하자.
- 비동기 방식의 처리가 필요한 경우 WebClient를 사용하자.
- HTTP Service를 interface로 추상화하여 더 유연하게 설계하고 싶다면 HTTP Interface를 사용하자.