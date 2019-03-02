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
		(blocking queue ma jakieś capacity, to że nie można dodać kolejnej rzeczy do kolejki nie powinien być sytuacją wyjątkową)
	* nie znalezienie encji w bazie danych wg specyfikacji JPA
		* https://docs.oracle.com/javaee/7/api/javax/persistence/EntityManager.html#getReference-java.lang.Class-java.lang.Object-
			* `EntityNotFoundException`
		* https://docs.oracle.com/javaee/7/api/javax/persistence/EntityManager.html#find-java.lang.Class-java.lang.Object-
			* to z kolei zwraca `null`, co też nie jest zbyt wygodne, bo potem wikłamy się w null-check
1. `Option` / `Optional` jako podejście do modelowania istnieje / nie istnieje
	* przewagi `Option` - bogatsze API
		* covariance (https://github.com/mtumilowicz/java11-covariance-contravariance-invariance) - chcemy supertyp
		    ```
			Option<String> a = Option.of("a");
			Option<CharSequence> narrowed = Option.narrow(a);
			```
		* Option jest izomorficzny z jednoelementową listą (albo posiada element albo jest pusty, więc powinien być 
		traktowany jako kolekcja)
			* `extends Iterable<T>`
			* Optional nie jest kolekcją
		* konwersja z listy `Option` na `Option` od listy wartości
			* `Option<Seq<String>> sequence = Option.sequence(List.of(Option.of("a"), Option.of("b")))`
			* `Option<Seq<String>> sequence = Option.sequence(List.of(Option.of("a"), Option.of("b"), Option.none()))`
		* `orElse` które przyjmuje `Option`
			* `Repository.findById(1).orElse(() -> Repository.findByName("Michal"))`
			* w javie 11 też to dodali (`or`): `findByName(person.getName()).or(() -> findById(person.getId())`
			* https://github.com/mtumilowicz/java11-optional
	* Option jest poprawie napisany (w sensie teorii kategorii) 
	(https://github.com/mtumilowicz/java11-category-theory-optional-is-not-functor)
		* z optionalem tak nie jest (map zmienia kontekst obliczeń - potrafi przełączyć na empty gdy funkcja zwraca 
		`null` - poważny błąd projektowy)
		    ```
			Function<Integer, Integer> nullFunction = i -> null;
			Function<Integer, String> toString = i -> nonNull(i) ? String.valueOf(i) : "null";
			Function<Integer, String> composition = nullFunction.andThen(toString);
			
			assertNotEquals(Optional.of(1).map(composition), Optional.of(1).map(nullFunction).map(toString));
			assertEquals(Optional.of(1).stream().map(composition).findAny(), Optional.of(1).stream().map(nullFunction).map(toString).findAny());
	        ```
	* https://github.com/mtumilowicz/java11-vavr-option
1. niestety nie wszytko da się zamodelować jako istnieje / nie istnieje i potrzebujemy bogatszego API (Try)
	* można myśleć o vavrowym `Try` jako o odpowiedniku try-catch-finally zencapsulowanym w obiekt
