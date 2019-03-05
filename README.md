# vavr-exceptions-handling-talk

1. wstęp
	* ile thread zajmuje w pamięci
	* co to stack
	* co to frame
	* https://github.com/mtumilowicz/java-stack
		* https://github.com/mtumilowicz/java8-stack-stackwalking
		* https://github.com/mtumilowicz/java9-stack-stackwalking
1. rzucanie exceptionow jest drogie
	* `fillInStackTrace`
	* stack unwinding
	* nie potrzebujemy zazwyczaj zrzutu calego stacku, tylko kilka (górnych) linii
	* koszt: koszt stworzenia wyjatku (niedeterministyczny - zalezny od wielkosci stacka) + stack unwinding
	* https://github.com/mtumilowicz/java11-exceptions-creating-exceptions-without-stacktrace
	* https://github.com/mtumilowicz/java11-exceptions-throwing-exceptions-is-expensive
1. wyjątki są nadużywane, a powinny modelować tylko sytuacje wyjątkowe
	* queue
		* (queue) `boolean add(E e)` - `IllegalStateException` if the element cannot be added at this time due to capacity restrictions
		* (blocking queue) `boolean offer(E e)` - returns true if the element was added to this queue, else false 
		(blocking queue ma jakieś capacity, to że nie można dodać kolejnej rzeczy do kolejki nie powinno być sytuacją wyjątkową)
	* nie znalezienie encji w bazie danych wg specyfikacji JPA
		* https://docs.oracle.com/javaee/7/api/javax/persistence/EntityManager.html#getReference-java.lang.Class-java.lang.Object-
			* `EntityNotFoundException`
		* https://docs.oracle.com/javaee/7/api/javax/persistence/EntityManager.html#find-java.lang.Class-java.lang.Object-
			* to z kolei zwraca `null`, co też nie jest zbyt wygodne, bo potem wikłamy się w null-check
1. `Option` / `Optional` jako podejście do modelowania istnieje / nie istnieje
	* przewagi `Option` - bogatsze API
		* Option jest izomorficzny z jednoelementową listą (albo posiada element albo jest pusty, więc powinien być 
		traktowany jako kolekcja)
			* `extends Iterable<T>`
			* Optional nie jest kolekcją
		* konwersja z listy `List<Option<T>>` na `Option<List<T>>`
			* `Option<Seq<String>> sequence = Option.sequence(List.of(Option.of("a"), Option.of("b")))`
			* `Option<Seq<String>> sequence = Option.sequence(List.of(Option.of("a"), Option.of("b"), Option.none()))`
		* `orElse` które przyjmuje `Option`
			* `Repository.findById(1).orElse(() -> Repository.findByName("Michal"))`
			* w javie 11 też to dodali (`or`): `findByName(person.getName()).or(() -> findById(person.getId())`
			* https://github.com/mtumilowicz/java11-optional
    * jest serializowany
	* `Option` jest poprawie napisany (w sensie teorii kategorii) 
	(https://github.com/mtumilowicz/java11-category-theory-optional-is-not-functor)
		* z optionalem tak nie jest (map zmienia kontekst obliczeń - przełącza kontekst na empty gdy funkcja zwraca 
		`null` - kolejny poważny błąd projektowy)
		    ```
			Function<Integer, Integer> nullFunction = i -> null;
			Function<Integer, String> toString = i -> nonNull(i) ? String.valueOf(i) : "null";
			Function<Integer, String> composition = nullFunction.andThen(toString);
			
			assertNotEquals(Optional.of(1).map(composition), Optional.of(1).map(nullFunction).map(toString));
			assertEquals(Optional.of(1).stream().map(composition).findAny(), Optional.of(1).stream().map(nullFunction).map(toString).findAny());
	        ```
	* https://github.com/mtumilowicz/java11-vavr-option
1. niestety nie wszystko da się zamodelować jako istnieje / nie istnieje i potrzebujemy bogatszego API (Try)
    * `Try` is a monadic container type which represents a computation 
      that may either result in an exception (`Throwable`), or return a successfully 
      computed value. Instances of `Try`, are either an instance of 
      `Success` or `Failure`
	* można myśleć o vavrowym `Try` jako o odpowiedniku try-catch-finally zencapsulowanym w obiekt
	* `interface Try<T>`
	    * `final class Success<T> implements Try<T>, Serializable`
	    * `final class Failure<T> implements Try<T>, Serializable`
	* parsing integer
	    ```
        Try<Integer> parseInteger = Try.of(() -> Integer.valueOf("a"));
        
        assertTrue(parseInteger.isFailure());
        ```
    * try with resources
        * java
            ```
            String fileName = "NonExistingFile.txt";
            
            String fileLines;
            try (var stream = Files.lines(Paths.get(fileName))) {
            
                fileLines = stream.collect(joining(","));
            }
            ```
        * vavr
            ```
            String fileName = "src/test/resources/lines.txt";
            Try<String> fileLines = Try.withResources(() -> Files.lines(Paths.get(fileName)))
                            .of(stream -> stream.collect(joining(",")));
            ```
            * success
                ```
                String fileName = "src/test/resources/lines.txt";
                
                Try<String> fileLines = Try.withResources(() -> Files.lines(Paths.get(fileName)))
                        .of(stream -> stream.collect(joining(",")));
                
                assertTrue(fileLines.isSuccess());
                assertThat(fileLines.get(), is("1,2,3"));
                ```
            * failure
                ```
                String fileName = "NonExistingFile.txt";
                
                Try<String> fileLines = Try.withResources(() -> Files.lines(Paths.get(fileName)))
                        .of(stream -> stream.collect(joining(",")));
                
                assertTrue(fileLines.isFailure());
                ```
    * https://github.com/mtumilowicz/java11-vavr-try
1. informacyjnie: try to tylko przelotka (i tak trzeba tworzyć te wyjątki, co jest kosztowne), więc może możnaby
opuścić założenie o tym, że Try ma albo sukces albo `Throwable`? Jest taka struktura - `Either` - to jest tak jakby
para, która ma albo lewą stronę albo prawą; zwyczajowo po prawej jest sukces a po lewej raport z porażki (konwencja)
1. 