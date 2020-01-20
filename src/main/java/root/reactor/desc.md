

``` java
Flux<String> seq1 = Flux.just("foo","bar","foobar");
// List 같이 생겼다

List<String> iterable = Arrays.asList("foo","bar","foobar");

Flux<String> seq2 = Flux.fromIterable(iterable);
// List -> Flux
```



``` java
Mono<String> noData = Mono.empty();
// 생성자 같다.

Mono<String> data = Mono.just("foo");

Flux<String> numbersFromFiveToSeven = Flux.range(5,3);

```

