# vavr-exceptions-handling-talk

1. preface
	* how much memory a thread takes in java
		* `1024KB`
		* `-Xss`
	* what is stack
	    * each thread in a JVM has its own JVM stack
		* holds local variables and partial results, and plays a part in method invocation and return
		* stores frames
	* what is frame
		* invocation context of a method
	* more precise info:
		* theory: https://github.com/mtumilowicz/java-stack
			* stackwalking on java 8: https://github.com/mtumilowicz/java8-stack-stackwalking
			* stackwalking on java 9: https://github.com/mtumilowicz/java9-stack-stackwalking
1. throwing exceptions is expensive
	* `fillInStackTrace` - records within this `Throwable` object information about the current state of the stack frames for the current thread
	* stack unwinding - process of destroying local objects and calling destructors (synonymous with the end of a function call and the subsequent popping of the stack)
	* general cost: creation cost (nondeterministic - depends on the stack size) + stack unwinding
	* usually we don't want to dump all the stack, but just a few top lines
	* it is possible to create exception without filling stacktrace: https://github.com/mtumilowicz/java11-exceptions-creating-exceptions-without-stacktrace
		* dedicated constructor
			```
			protected Exception(String message, Throwable cause,
					    boolean enableSuppression,
					    boolean writableStackTrace) {
			    super(message, cause, enableSuppression, writableStackTrace);
			}
			```
			```
			class ExceptionWithoutStackTrace extends RuntimeException {
			    ExceptionWithoutStackTrace() {
				super(null, null, false, false);
			    }
			}
			```
		* overridding `fillInStackTrace` method
			```
			class ExceptionWithOverriddenFillInStackTrace extends RuntimeException {

			    @Override
			    public synchronized Throwable fillInStackTrace() {
				return this;
			    }
			}
			```
	* measured with JMH: https://github.com/mtumilowicz/java11-exceptions-throwing-exceptions-is-expensive
		```
		Benchmark                        Mode  Cnt         Score          Error  Units
		Jmh.exceptionWithStackTrace     thrpt    4    657794,785 ±    48723,978  ops/s
		Jmh.exceptionWithoutStackTrace  thrpt    4  68883171,304 ± 19209621,332  ops/s
		Jmh.runtimeException            thrpt    4    600517,626 ±   686481,438  ops/s
		```
1. exceptions are overused - there should model only exceptional behaviours
	* queue
		* (queue) `boolean add(E e)` - `IllegalStateException` if the element cannot be added at this time due to capacity restrictions
		* (blocking queue) `boolean offer(E e)` - returns true if the element was added to this queue, else false;
		when using a capacity-restricted queue, this method is generally preferable to add, which can fail to insert 
		an element only by throwing an exception (because in bounded queues it is not an exceptional behaviour)
	* JPA specification - entity cannot be found in the database
		* `<T> T getReference(Class<T> entityClass, Object primaryKey)`
			* throws `EntityNotFoundException`
		* `<T> T find(Class<T> entityClass, Object primaryKey)`
			* returns `null`, so we are involved in further null-checks or `NullPointerException`
1. `Option` as a way of modelling exists / not exists
	* bigger, more flexible API than `Optional`
		* `extends Iterable<T>` - `Option` is isomorphic to singleton list (either has element or not, so it could be treated as collection)
		* conversion `List<Option<T>> -> Option<List<T>>`
			* `Option<Seq<String>> sequence = Option.sequence(List.of(Option.of("a"), Option.of("b")))`
		* `orElse` supplier of `Option`
			* `CacheRepository.findById(1).orElse(() -> DatabaseRepository.findById(1))`
    	* `Serializable`
	* **workshops**: https://github.com/mtumilowicz/java11-vavr093-option-workshop
1. not everything could be modelled as exists / not exists - we need more flexible API
    * `Try` is a monadic container type which represents a computation 
      that may either result in an exception (`Throwable`), or return a successfully 
      computed value. Instances of `Try`, are either an instance of 
      `Success` or `Failure`
    * you can think about `Try` as a pair `(Failure, Success) ~ (Throwable, Object)` 
        that has either left or right value
	* you can think about `Try` as an object representation of try-catch-finally 
	* parsing integer
	    * success
            ```
            Try<Integer> parseInteger = Try.of(() -> Integer.valueOf("1"));
            
            assertTrue(parseInteger.isSuccess());
            assertThat(parseInteger.get(), is(1));
            ```
        * failure
            ```
            Try<Integer> parseInteger = Try.of(() -> Integer.valueOf("a"));
    
            assertTrue(parseInteger.isFailure());
            assertTrue(parseInteger.getCause() instanceof NumberFormatException);
            assertThat(parseInteger.getCause().getMessage(), is("For input string: \"a\""));
            ```
    * try with resources
        * java
            ```
            String fileLines;
            try (var stream = Files.lines(Paths.get(fileName))) {
            
                fileLines = stream.collect(joining(","));
            }
            ```
        * vavr
            ```
            Try<String> fileLines = Try.withResources(() -> Files.lines(Paths.get(fileName)))
                            .of(stream -> stream.collect(joining(",")));
            ```
    * **workshops**: https://github.com/mtumilowicz/java11-vavr093-try-workshop
1. Digression: `Try` is just a very handy wrapper - we still have to create exceptions and deal with its cost - maybe there is
a structure that `(Object, Object)` with convention that on the left side we have failure and on the right - success?
That structure is called `Either`.
    * workshops (?)
1. Function lifting
    * partial function from `X` to `Y` is a function `f: X′ → Y`, 
                       for some `X′ c X`. For `x e X\X′` function is undefined
    * in programming - if partial function is called with a disallowed 
                       input value, it will typically throw an exception
    * in programming - we **lift** function `f` to `f′: X -> Option<Y>` in such a manner:
        * `f′(x).get() = f(x)` on `X′`
        * `f′(x) = Option.none()` for `x e X\X′`
    * lifting function with `Option`
        ```
        Function2<Integer, Integer, Integer> divide = (a, b) -> a / b;
        
        Function2<Integer, Integer, Option<Integer>> lifted = Function2.lift(divide);
        ```
    * lifting function with `Try`
        ```
        Function2<Integer, Integer, Integer> divide = (a, b) -> a / b;
        
        Function2<Integer, Integer, Try<Integer>> lifted = Function2.liftTry(divide);
        ```
    * **workshops**: https://github.com/mtumilowicz/java11-vavr093-partial-function-lifting-workshop
1. Mentioned earlier API: `Option`, `Try`, `Either`, function lifting - are not applicable to objects validation
    * no easy way to aggregate errors
    * computations are broken after first failure
    * we need a structure that is so-called applicative
    * vavr `Validation` control is an applicative functor and facilitates accumulating errors
        * we want to validate incoming request:
            ```
            @Builder
            @Value
            class PersonRequest {
                String name;
                AddressRequest address;
                List<String> emails;
                int age;
            }
            ```
        * if valid, we want to return:
            ```
            @Builder
            @Value
            public class ValidPersonRequest {
                Word name;
                ValidAddressRequest address;
                Emails emails;
                Age age;
            }
            ```
        * if not valid, we want to return aggregated errors: for example `Seq<String>`
        * Validator
            ```
            public class PersonRequestValidation {
                public static Validation<Seq<String>, ValidPersonRequest> validate(PersonRequest request) {
            
                    return Validation
                            .combine(
                                    Word.validate(request.getName()),
                                    Email.validate(request.getEmails()).mapError(error -> error.mkString(", ")),
                                    AddressRequestValidation.validate(request.getAddress()).mapError(error -> error.mkString(", ")),
                                    NumberValidation.positive(request.getAge()))
                            .ap((name, emails, address, age) -> ValidPersonRequest.builder()
                                    .name(Word.of(name))
                                    .emails(emails.map(Email::of).transform(Emails::new))
                                    .address(address)
                                    .age(Age.of(age))
                                    .build());
                }
            }
            ```
        * we could easily send invalid request to the `PatchService`
        * we dont need any infrastructure providers (AOP, dynamic proxies, DI)
        * only 8 slots in combine
    * JSR303: https://github.com/mtumilowicz/java11-jsr303-custom-validation (do we need workshops?)
        * entity
            ```
            @Value
            @Builder
            public class User {
            
                @NotBlank
                @Word
                String name;
            
                @JsonCreator(mode = JsonCreator.Mode.PROPERTIES)
                public User(@JsonProperty("name") String name) {
                    this.name = name;
                }
            }
            ```
        * annotation
            ```
            @Constraint(validatedBy = WordValidator.class)
            @Target(ElementType.FIELD)
            @Retention(RetentionPolicy.RUNTIME)
            public @interface Word {
                String message() default "is not a proper word!";
                Class<?>[] groups() default {};
                Class<? extends Payload>[] payload() default {};
            }
            ```
        * validator
            ```
            class WordValidator implements ConstraintValidator<Word, String> {
            
                private static final Predicate<String> PATTERN = Pattern.compile("[\\w]+").asMatchPredicate();
            
                @Override
                public void initialize(Word word) {
                }
            
                @Override
                public boolean isValid(String word,
                                       ConstraintValidatorContext cxt) {
                    return isNull(word) || PATTERN.test(word);
                }
            
            }
            ```
        * endpoint
            ```
            @RestController
            @RequestMapping("/users")
            public class UserController {
                
                @PostMapping("register")
                public ResponseEntity<User> register(@RequestBody @Valid User user) {
                    return ResponseEntity.ok(user);
                }
            }
            ```
        * bad request is rejected and it is hard to intercept it in a reasonable way
        * we need infrastructure providers (AOP, dynamic proxies, DI)
        * instead of creating domain objects we create annotations - objects still could be created in an invalid
        state
    * **workshops**: https://github.com/mtumilowicz/java11-vavr093-validation-workshop
