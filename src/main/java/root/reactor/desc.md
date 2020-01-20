

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

//

``` java
public static void main(String[] args){
    final List<String> basket1 = Arrays.asList(new String[]{"kiwi", "orange", "lemon", "orange", "lemon", "kiwi"});
    final List<String> basket2 = Arrays.asList(new String[]{"banana", "lemon", "lemon", "kiwi"});
    final List<String> basket3 = Arrays.asList(new String[]{"strawberry", "orange", "lemon", "grape", "strawberry"});
    final List<List<String>> baskets = Arrays.asList(basket1,basket2,basket3);
    final Flux<List<String>> basketFlux = Flux.fromIterable(baskets);
}
```

//

``` java
// 값을 꺼내서 새로운 Publisher 로 바꾸어준다!
basketFlux.concatMap(basket -> {
    final Mono<List<String>> distinctFruits = Flux.fromIterable(basket).distinct().collectList();
    final Mono<Map<String,Long>> countFruitsMono = Flux.fromIterable(basket)
        .groupBy(fruit -> fruit)
        .concatMap(groupedFlux ->
        	groupedFlux.count()
                   .map(count -> {
                       final Map<String,Long> fruitCount = new LinkedHashMap<>();
                       fruitCount.put(groupedFlux.key(),count);
                       return fruitCount;
                   })
        ).reduce((accumulatedMap,currentMap) ->
        	new LinkedHashMap<String,Long>(){{
                putAll(accumulatedMap);
                putAll(currentMap);
            }}        
        );
    return Flux.zip(distinctFruits,countFruitsMono, (distinct,count) -> new FruitInfo(distinct,count));
}).subscribe(System.out::println);
```

//

``` java
public class FruitInfo{
    private final List<String> distinctFruits;
    private final Map<String,Long> countFruits;
    
    public FruitInfo(List<String> distinctFruits, Map<String,Long> countFruits){
        this.distinctFruits=distinctFruits;
        this.countFruits=countFruits;
    }
    
    @Override
    public boolean equals(Object o){
        if(this == o)
            return true;
        if(o == null || getClass() != o.getClass())
            return false;
        
        FruitInfo fruitInfo = (FruitInfo) o;
        if(distinctFruits != null? !distinctFruits.equals(fruitInfo.distinctFruits) : fruitInfo.distinctFruits != null)
            return false;
        return countFruits != null ? countFruits.equals(fruitInfo.countFruits) : fruitInfo.countFruits == null;
    }
    
    @Override
    public int hashCode(){
        int result = distinctFruits != null ? distinctFruits.hashCode() : 0;
        result = 31 * result + (countFruits != null ? countFruits.hashCode() : 0);
        return result;
    }
    
    @Override
    public String toString(){
        return "FruitInfo{"+
            "distinctFruits="+distinctFruits +
            ", countFruits="+countFruits+
            "}";
    }
}
```

//

``` java
basket.concatMap(basket -> {
    final Mono<List<String>> distinctFruits = Flux.fromIterable(basket).distinct().collectList().subscribeOn(Schedulers.parallel());
    final Mono<Map<String,Long>> countFruitsMono = Flux.fromIterable(basket)
        .groupBy(fruit -> fruit)
        .concatMap(groupedFlux ->
        	groupedFlux.count()
                   .map(count -> {
                       final Map<String,Long> fruitCount = new LinkedHashMap<>();
                       fruitCount.put(groupedFlux.key(),count);
                       return fruitCount;
                   })
        ).reduce((accumulatedMap,currentMap) -> 
            new LinkedHashMap<>(){{     
        		putAll(accumulatedMap);
            	putAll(currentMap);
        	}}
        ).subscribeOn(Schedulers.parallel());
    return Flux.zip(distinctFruits.countFruitsMono, (distinct,count) -> new FruitInfo(distinct,count));
}).subscribe(System.out::println);
```

//

``` java
basketFlux.concatMap(basket -> {
    // ... 생략
    return Flux.zip(distinctFruits,countFruitsMono,(distinct,count) -> new FruitInfo(distinct,count));
}).subscribe(
	System.out::println(error);
    error -> {
        System.err.println(error);
        countDownLatch.countDown();
    },
    () -> {
        System.out.println("complete");
        countDownLatch.countDown();
    }
);
countDownLatch.await(2, TimeUnit.SECONDS);
```

//

``` java
Flux.fromIterable(basket).log()...
Flux.fromIterable(basket).log()...    
```

//

``` java
basketFlux.concatMap(basket -> {
    final Flux<String> source = Flux.fromIterable(basket).log().publish().autoConnect(2);
    final Mono<List<String>> distinctFruits = source.distinct().collectList();
    final Mono<Map<String,Long>> countFruitsMono = source.groupBy(fruit -> fruit)
        .concatMap(groupedFlux -> groupedFlux.count()
                   .map(count -> {
                       final Map<String,Long> fruitCount = new LinkedHashMap<>();
                       fruitCount.put(groupedFlux.key(),count);
                       return fruitCount;
                   })
        ).reduce((accumulatedMap,currentMap) -> new LinkedHashMap<String,Long>() {{
        putAll(accumulatedMap);
        putAll(currentMap);
    }});
    return Flux.zip(distinctFruits,countFruitsMono,(distinct,count) -> new FruitInfo(distinct,count));
}).subscribe(
	System.out::println,
    error -> {
        System.err.println(error);
        countDownLatch.countDown();
    },
    () -> {
        System.out.println("complete");
        countDownLatch.countDown();
    }
)
```

//

``` java
source.publishOn(Schedulers.parallel())...
```

//

``` java
@Test
public void testFruitBaskets(){
    final FruitInfo expected1 = new FruitInfo(
    	Arrays.asList("kiwi","orange","lemon"),
        new LinkedHashMap<String,Long>(){{
            put("kiwi",2L);
            put("orange",2L);
            put("lemon",2L);
        }}
    );
    StepVerifier.create(getFruitsFlux())
        .expectNext(expected1)
        .verifyComplete();
}
```

