# vavr-exceptions-handling-talk

1. wstęp
	* ile thread zajmuje w pamięci
	* co to stack
	* co to frame
	* https://github.com/mtumilowicz/java-stack
		* https://github.com/mtumilowicz/java8-stack-stackwalking
		* https://github.com/mtumilowicz/java9-stack-stackwalking
1. rzucanie exceptionow jest drogie
	* fillInStackTrace
	* stack unwinding
	* nie potrzebujemy zazwyczaj zrzutu calego stacku, tylko kilka (górnych) linii
	* koszt: koszt stworzenia wyjatku (niedeterministyczny - zalezny od wielkosci stacka) + stack unwinding
	* https://github.com/mtumilowicz/java11-exceptions-creating-exceptions-without-stacktrace
	* https://github.com/mtumilowicz/java11-exceptions-throwing-exceptions-is-expensive
1. wyjątki są nadużywane, a powinny modelować tylko sytuacje wyjątkowe
	* queue
		* (queue) boolean add(E e) - IllegalStateException if the element cannot be added at this time due to capacity restrictions
		* (blocking queue) boolean offer(E e) - returns true if the element was added to this queue, else false (blocking queue ma jakieś capacity,
			to że nie można dodać kolejnej rzeczy do kolejki nie powinien być sytuacją wyjątkową)
	* nie znalezienie encji w bazie danych wg specyfikacji JPA
		* https://docs.oracle.com/javaee/7/api/javax/persistence/EntityManager.html#getReference-java.lang.Class-java.lang.Object-
			* EntityNotFoundException
		* https://docs.oracle.com/javaee/7/api/javax/persistence/EntityManager.html#find-java.lang.Class-java.lang.Object-
			* to z kolei zwraca nulla, co też nie jest zbyt wygodne, bo potem wikłamy się w null-check
1. Option / Optional jako podejście do modelowania istnieje / nie istnieje
	
	* https://github.com/mtumilowicz/java11-vavr-option
	* https://github.com/mtumilowicz/java11-category-theory-optional-is-not-functor
