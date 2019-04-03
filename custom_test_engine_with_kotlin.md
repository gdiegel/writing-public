# Writing a custom test engine with Kotlin

### Introduction

Among the many new concepts (see my other blog articles for an overview) introduced in JUnit5 is the project's [modular composition][1]. The JUnit platform module serves as a foundation for launching testing frameworks on the JVM. It also defines The [TestEngine][2] API, which allows you to plug in your own implementation. In this blog post we want to explore how we can write our own custom TestEngine to develop a testing framework which runs on the platform. For a little twist we are going to use the fairly new [Kotlin][3] programming language to write the test engine.

### The TestEngine API

The `org.junit.platform.engine.TestEngine` interface describes an API to discover and execute tests on the JUnit platform. JUnit provides the Jupiter implementation of this API to discover and execute tests written with the Jupiter programming model. It also provides a `org.junit.vintage.engine.VintageTestEngine` implementation to launch tests written for JUnit4. We can create our own `TestEngine` by implementing the `TestEngine` interface.The two most important methods in `org.junit.platform.engine.TestEngine` are `discover` and `execute`:

```java
	TestDescriptor discover(EngineDiscoveryRequest discoveryRequest, UniqueId uniqueId);

	void execute(ExecutionRequest request);
```

`discover` takes an `org.junit.platform.engine.EngineDiscoveryRequest` and a `org.junit.platform.engine.UniqueId` and returns a TestDescriptor while `execute` takes an `org.junit.platform.engine.ExecutionRequest` but has no return type. The `org.junit.platform.engine.EngineDiscoveryRequest` contains various filters and selectors to determine the to-be-executed tests and used engines, the `org.junit.platform.engine.UniqueId` allows us to pass a unique identifier for this specific `org.junit.platform.engine.EngineDiscoveryRequest`.

 ```kotlin
 class KionEngine : TestEngine {

    private val isKionContainer = Predicate<Class<*>> { it.superclass == KionSpec::class.java }

    override fun discover(discoveryRequest: EngineDiscoveryRequest, uniqueId: UniqueId): TestDescriptor {
        val classNamePredicate = buildClassNamePredicate(discoveryRequest)
        val testDescriptor = EngineDescriptor(uniqueId, id)

        discoveryRequest.getSelectorsByType(PackageSelector::class.java).forEach { selector ->
            findAllClassesInPackage(selector.packageName, isKionContainer, classNamePredicate).forEach { testClass ->
                println("Adding class ${testClass.name}")
                testDescriptor.addChild(KionClass(testClass, testDescriptor))
            }
        }

        return testDescriptor
    }
 ```

### Implementing the API

In our custom `KionEngine` (Kion being the Peruvian word for [ginger][4]) we keep it simple and allow only packages to be searched for relevant test classes. We use reflection to find all valid test classes, valid being those who satisfy the predicate `isKionContainer` and are subclasses of `KionSpec`. We then go through each testClass and add a new instance of `KionClass` constructed from this very class to the testDescriptor.

```kotlin
class KionClass(
        klass: Class<*>,
        parent: TestDescriptor
) : AbstractTestDescriptor(
        parent.uniqueId.append("class", klass.name),
        klass.simpleName,
        ClassSource.from(klass)
) {
    val isKionUnit = Predicate<Method> { AnnotationUtils.isAnnotated(it, Kion::class.java) }

    init {
        this.klass = klass
        setParent(parent)
        addAllChildren()
    }

    private fun addAllChildren() {
        findMethods(klass, isKionUnit).stream()
                .map { method -> KionUnit(method, klass, this) }
                .forEach { this.addChild(it) }
    }

    override fun getType(): TestDescriptor.Type = TestDescriptor.Type.CONTAINER
}
```

`KionClass` inherits from `org.junit.platform.engine.support.descriptor.AbstractTestDescriptor` (i.e. "is a" type of `TestDescriptor`). Upon initialization, all functions annotated with our `@Spec` annotation are considered test units, mapped to `KionUnit` and added to the testDescriptor.

```kotlin
class KionUnit(method: Method, klass: Class<*>, parent: KionClass) : AbstractTestDescriptor(
        parent.getUniqueId().append("method", method.getName()),
        method.name,
        MethodSource.from(method)
) {

    internal val method: Method
    internal val klass: Class<*>

    init {
        setParent(parent)
        this.method = method
        this.klass = klass
    }

    override fun getType(): TestDescriptor.Type = TestDescriptor.Type.TEST
}
```

A `KionUnit` represents an actual test unit or test function and also inherits from `org.junit.platform.engine.support.descriptor.AbstractTestDescriptor`. The implementation is rather straight forward, the corresponding function and enclosing class are passed in the constructor. Note that the code uses Java's reflection system and so speaks of methods and uses the class `java.lang.reflect.Method`. Functions in Kotlin have broader scope since they don't necessarily have to be associated with an object. The may exist as first-class or top-level functions (i.e. outside the scope of a class).

All test classes extend from a common base class that contains a single `spec` function:

```kotlin
open class KionSpec {

    fun spec(description: String?, body: () -> Unit) {
        println("Spec $description")
        try {
            body()
        } finally {
            println("Exiting spec block")
        }
    }
}
```

The `spec` function is a [higher-order function][5] (meaning that it can take other functions as arguments) and we can pass a `String` description and an instance of a function type as an argument. It will accept no arguments and have the return type `Unit`. The description allows to briefly describe the scope and intention of the test and the second argument will contain the actual test code. With this base class we can now construct our actual test class:

```kotlin
class MySpec : KionSpec() {

    @Kion
    fun will_pass() {
        spec("a should be equal to a",
                {
                    assertThat("a").isEqualTo("a")
                }
        )
    }

    @Kion
    fun will_fail() {
        spec("a should be equal to b",
                {
                    assertThat("a").isEqualTo("b")
                }
        )
    }
}
```

Test functions have to be annotated with the `@Kion` annotation to be considered test by the test engine. Compare that to JUnit's `@Test` annotation, which serves a similar purpose. Both tests use the `spec` function, pass in a short description and use lambdas containing assertion statements to [instantiate][6] the function type. In this case we have two test functions, one passing and one failing

### Using the platform launcher to execute the tests

Executing the tests means getting the root test descriptor (i.e. the actual test engine), recursively traversing all children `TestDescriptor`s and executing if they are of type `KionUnit` (remember, `KionClass` and `KionUnit` have the super type `TestDescriptor`):

```kotlin
    override fun execute(request: ExecutionRequest) {
        val engine = request.rootTestDescriptor
        val listener = request.engineExecutionListener
        listener.executionStarted(engine)
        engine.children.forEach { child -> execute(child, listener) }
        listener.executionFinished(engine, successful())
    }

    private fun execute(descriptor: TestDescriptor, listener: EngineExecutionListener) {
        listener.executionStarted(descriptor)
        when (descriptor) {
            is KionClass -> descriptor.children.forEach { child -> execute(child, listener) }
            is KionUnit -> {
                val result = executeKionUnit(descriptor)
                listener.executionFinished(descriptor, result)
            }
        }
    }

    private fun executeKionUnit(child: KionUnit): TestExecutionResult {
        val instance = ReflectionUtils.newInstance(child.klass)
        try {
            ReflectionUtils.invokeMethod(child.method, instance)
        } catch (e: Throwable) {
            return failed(e)
        }
        return successful()
    }
```

We once more use Java's reflection system to create a new instance of the enclosing test class and invoke the test function. The lowest level function returns a `org.junit.platform.engine.TestExecutionResult` to be recorded by the listener. All that's left to do now is create a provider configuration file in `src/main/resources/META-INF/services` to identify our test engine as a service provider to Java's service loader mechanism. We can then use JUnit5's platform launcher to start our test suite:

```kotlin
fun main(args: Array<String>) {
    val request = LauncherDiscoveryRequestBuilder.request()
            .selectors(
                    selectPackage("io.example")
            )
            .build()

    val launcher = LauncherFactory.create()
    val summaryGeneratingListener = SummaryGeneratingListener()
    launcher.registerTestExecutionListeners(summaryGeneratingListener)
    launcher.execute(request)
    summaryGeneratingListener.summary.printTo(PrintWriter(System.out))
}
```

Running the class produces the following output:

```
Adding class io.example.MySpec
KionUnit(method=public final void io.example.MySpec.will_fail(), klass=class io.example.MySpec)
KionUnit(method=public final void io.example.MySpec.will_pass(), klass=class io.example.MySpec)
Starting container MySpec
Starting container will_fail
Executing KionUnit(method=public final void io.example.MySpec.will_fail(), klass=class io.example.MySpec)
Spec a should be equal to b
Container will_fail returned TestExecutionResult [status = FAILED, throwable = org.opentest4j.AssertionFailedError: 
Expecting:
 <"a">
to be equal to:
 <"b">
but was not.]
Starting container will_pass
Executing KionUnit(method=public final void io.example.MySpec.will_pass(), klass=class io.example.MySpec)
Spec a should be equal to a
Container will_pass returned TestExecutionResult [status = SUCCESSFUL, throwable = null]

Test run finished after 183 ms
[         2 containers found      ]
[         0 containers skipped    ]
[         2 containers started    ]
[         0 containers aborted    ]
[         1 containers successful ]
[         0 containers failed     ]
[         2 tests found           ]
[         0 tests skipped         ]
[         2 tests started         ]
[         0 tests aborted         ]
[         1 tests successful      ]
[         1 tests failed          ]
```

This implementation shows how to use JUnit's API to write a custom test engine. However, it obviously follows JUnit's standard test design very closely. Test functions have to be annotated to be recognized as such and we make heavy use of the provided launcher architecture. The base test class also feels quite restrictive, as it offers only one function to use in writing tests. We will explore a more free-flowing test design and an evolved test engine architecture at a later time. You can browse the full sources [here][7].

[1]: https://junit.org/junit5/docs/current/user-guide/#overview-what-is-junit-5 "junit-5"
[2]: https://junit.org/junit5/docs/current/api/org/junit/platform/engine/TestEngine.html "test-engine"
[3]: https://kotlinlang.org "kotlin"
[4]: https://en.wikipedia.org/wiki/Ginger "ginger"
[5]: https://kotlinlang.org/docs/reference/lambdas.html#higher-order-functions "higher-order-functions"
[6]: https://kotlinlang.org/docs/reference/lambdas.html#instantiating-a-function-type "instantiating-a-function-type"
[7]: https://github.com/gdiegel/kion
