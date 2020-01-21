`WebClient client1 = WebClient.create();`

`WebClient client2 = WebClient.create("http://localhost:8080);`



``` java
WebClient client3 = WebClient.builder()
    .baseUrl("http://localhost:8080")
    .defaultCookie("coolieKey", "cookieValue")
    .defaultHeader(HttpFeaders.CONTENT_TYPE, MediaType.APPLICATION_JSON_VALUE)
    .defaultUriVariables(Collections.singletonMap("url","http://localhost:8080"))
    .build();
```



preparing a request

``` java
WebClient.UriSpec<WebClient.RequestBodySpec> request1 = client3.method(HttpMethod.POST);

WebClient.UriSpec<WebClient.RequestBodySpec> request2 = client2.post(); // 이게 더 쉽네!
```



``` java
WebClient.RequestBodySpec uri1 = client3.method(HttpMethod.POST)
    .url("/resource");

WeebClient.RequestBodySpec uri2 = client3.post()
    .uri(URI.create("/resource"));
```



``` java
WebClient.RequestHeadersSpec requestSpec1 = WebClient.create()
    .method(HttpMethod.POST)
    .uri("/resource")
    .body(BodyInserters.fromPublisher(Mono.just("data")), String.class); // Hot ?

WebClient.RequestHeadersSpec<?> requestSpec2 = WebClient.create("http...")
    .post()
    .uri(URI.create("/resource"))
    .body(BodyInserters.fromObject("data"));
```



``` java
BodyInserter<Publisher<String>, ReactiveHttpOutputMessage> inserter1 = BodyInserters.fromPublisher(Subscriber::onComplete, String.class);

// MultiValueMap
LinkedMultiValueMap map = new LinkedMultiValueMap();
map.add("key1","value1");
map.add("key2","value2");
BodyInserter<MultiValueMap, ClientHttpRequest> inserter2 = BodyInserters.fromMultipartData(map);

// single object
BodyInserter<Object,ReactiveHttpOutputMessage> inserter3 = BodyInserters.fromObject(new Object());
```



``` java
WebClient.ResponseSpec response1 = uri1.body(inserter3)
    .header(HttpHeaders.CONTENT_TYPE,MediaType.APPLICATION_JSON_VALUE)
    .accept(MediaType.APPLICATION_JSON, MediaType.APPLICATION_XML)
    .acceptCharset(Charset.forName("UTF-8"))
    .ifNoneMatch("*")
    .ifModifiedSine(ZonedDataTime.now())
    .retrieve();
```



getting a response

``` java
String response2 = request1.exchange()
    .block()
    .bodyToMono(String.class)
    .block();

String response3 = request2.retrieve()
    .bodyToMono(String.class)
    .block();
```



WebTestClient

``` java
WebTestClient testClient = WebTestClient.bindToServer()
    .baseUrl("http://localhost:8080")
    .build();
```



``` java
RouterFunction function = RouterFunctions.route(
	RequestPredicates.GET("/resource"),
    request -> ServerResponse.ok().build()
);

WebTestClient.bindToRouterFunction(function)
    .build().get().uri("/resource")
    .exchange()
    .expectStatus().isOk()
    .expectBody.isImpty();
```



``` java
WebHandler handler = exchange -> Mono.empty();
WebTestClient.bindToWebHandler(handler).build();
```



``` java
@Autowired
private ApplicationContext context;

WebTestClient testClient = WebTestClient.bindToApplicationContext(context);
```



``` java
@Autowired
private Controller controller;

WebTestClient testClient = WebTestClient.bindToController(controller).build();
```



``` java
WebTestClient.bindToServer()
    .baseUrl("http://localhost:8080")
    	.build()
    	.post()
    	.uri("/resource")
    .exchange()
    	.expectStatus().isCreated()
    	.expectHeader().valueEquals("Content-Type", "application/json")
    	.expectBody().isEmpty();
```



