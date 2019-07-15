# JUnit5 -- Feeble screams from forests well known

## Introduction

This article aims to serve as a quick guide to the (in my opinion) best new features of JUnit5 but also some of the obstacles to overcome on the way. Development on JUnit5 began roughly ten years after the release of JUnit4, one of the most popular libraries in the Java world. The third milestone M3 of JUnit5 was released on November 30, 2016 with the final release being scheduled for Q1 2017.

I'm not going to focus on obvious changes like renamed annotations (`@BeforeClass` --> `@BeforeAll`) or test class no longer having to be public. These have been much written about and can be looked up in the official guide.

## History & Architecture

JUnit5 will arrive more than ten years after the release of JUnit4. Unlike Junit4, it will not be a monolithic library but come in modules.

`JUnit 5 = JUnit Jupiter + JUnit Platform + JUnit Vintage`

JUnit Jupiter is what developers will be most in contact with; It consists of the following modules:

* An API for developers to write tests against (`junit-jupiter-api`)
* An engine for each API to discover, present and run the corresponding tests (`junit-jupiter-engine`)

The JUnit Platform serves as the foundation for launching testing frameworks on the JVM. It also defines a TestEngine API to let developers plug into the JUnit platform's launching infrastructure. It's made up of:

* An API that all engines have to implement so they can be used uniformly (`junit-platform-engine`)
* A mechanism that orchestrates the engines (`junit-platform-launcher`)

JUnit Vintage is an implementation of the JUnit platform engine that runs legacy tests and consists of one module:

* An engine to run tests written against the JUnit3 and JUnit4 APIs (`junit-vintage-engine`)

## Migrating from JUnit4

Using Junit5 requires one dependency for the easiest use cases:

```xml
<dependency>
    <groupId>org.junit.jupiter</groupId>
    <artifactId>junit-jupiter-engine</artifactId>
    <version>5.0.0-M3</version>
    <scope>test</scope>
</dependency>
```

There is presently no full support of JUnit5 by the `maven-surefire-plugin`, so in order to run tests by invoking Surefire you need to plug in the provider developed by the JUnit team:

```xml
<build>
    <plugins>
        <plugin>
            <artifactId>maven-surefire-plugin</artifactId>
            <dependencies>
                <dependency>
                    <groupId>org.junit.platform</groupId>
                    <artifactId>junit-platform-surefire-provider</artifactId>
                    <version>1.0.0-M3</version>
                </dependency>
            </dependencies>
        </plugin>
    </plugins>
</build>
```

The same works for the `maven-failsafe-plugin`, however the provider is pretty crude and doesn't support all configuration options of Surefire or Failsafe. Writing a JUnit5 test is pretty straightforward:

```java
import org.junit.jupiter.api.Test;
import static org.assertj.core.api.Assertions.assertThat;

class RetryTest {

    @Test
    void canRetryOnce() {
        Integer i = 0;
        final Retry<Integer> retry = Retry.<Integer>builder().withRetries(1).build();
        assertThat(retry.call(() -> i + 1)).isEqualTo(1);
    }
}
```

Since my test project containes only JUnit5 tests, it's only the `junit-jupiter-engine` that will be launched:

`Dec 22, 2016 4:14:49 PM org.junit.platform.launcher.core.ServiceLoaderTestEngineRegistry loadTestEngines INFO: Discovered TestEngines with IDs: [junit-jupiter]`

Legacy JUnit4 tests may be run with the `junit-vintage-engine`:

```xml
<dependency>
    <groupId>org.junit.vintage</groupId>
    <artifactId>junit-vintage-engine</artifactId>
    <version>4.12.0-M3</version>
</dependency>
```

This allows you to migrate your tests step by step. You may also write new tests with Junit5 while running legacy JUnit4 tests without having to change anything about them. Just include both dependencies in your project.

## Extensions

By extending `BlockJUnit4ClassRunner`, developers are able to create custom runners in JUnit4. However, doing so often involved overriding lot's of methods of the parent class. The later introduced `@Rule` concept provided a certain aspect of composability, but always felt “tacked-on” instead of being part of a larger concept. One of several major improvements in JUnit5 is the extension model. In contrast to `Runner`, `@Rule`, and `@ClassRule`, the extension model consists of a single, coherent concept: the Extension API. If you are currently using the `@RunWith` annotation to invoke a specific class to run your tests instead of the built-in runner, or use the `@Rule` annotation, you'll have to instead use an extension. `junit-jupiter-api` supplies many implementations of the Extension interface, e.g. `TestExecutionCondition`:

```java
@FunctionalInterface
@API(Experimental)
public interface TestExecutionCondition extends Extension {
   
    ConditionEvaluationResult evaluate(TestExtensionContext context);
}
```

Interfaces extending `Extension` take a `TestExtensionContext` or `ContainerExecutionContext` as the parameter of their singular method. A container is anything that contains tests, e.g. a class, an interface, etc. Implementing your own extensions is very easy: E.g. to only run your tests in stages denoted by a `@Stageannotation` you would implement `TestExecutionCondition` as follows:

```java
import com.unitedinternet.portal.qa.lib.resource.Stage;
import com.unitedinternet.portal.qa.lib.resource.Stages;
import org.junit.jupiter.api.extension.ConditionEvaluationResult;
import org.junit.jupiter.api.extension.TestExecutionCondition;
import org.junit.jupiter.api.extension.TestExtensionContext;
import org.junit.platform.commons.util.AnnotationUtils;
import java.lang.reflect.AnnotatedElement;
import java.util.Optional;

class StageMatchesCondition implements TestExecutionCondition {
 
    Stage readCurrentStage(){
        // read current stage from ENV, properties or otherwise...
    }

    @Override
    public ConditionEvaluationResult evaluate(TestExtensionContext context) {
        if (elementMatches(context.getElement())){
            return ConditionEvaluationResult.enabled("Stage matches annotation");
        }
        return ConditionEvaluationResult.disabled(format("Test not annotated to run in stage [%s]", readStage()));
    }

    private boolean elementMatches(Optional<AnnotatedElement> element) {
        return AnnotationUtils.findAnnotation(element, Stages.class)
            .filter(stages -> asList(stages.value())
            .contains(readStage())) .isPresent();
    }
}
```

Use the `@ExtendWith` annotation to register your custom extension:

```java
import org.junit.jupiter.api.extension.ExtendWith;
import com.unitedinternet.portal.qa.lib.resource.Stages;
import static com.unitedinternet.portal.qa.lib.resource.Stage.PRELIVE;
import static com.unitedinternet.portal.qa.lib.resource.Stage.LIVE;

@ExtendWith(StageMatchesCondition.class)
@Stages(Stage.PRELIVE, Stage.LIVE)
class PreLiveAndLiveOnlyIT(){
    [...]
}
```

Extensions have the advantage of being repeatable, and thus make for composable tests. Refer to the official guide for an overview of the supplied extensions.

## Platform Launcher API

JUnit4's interface between JUnit and programmatic clients has been the class `JUnitCore`. This class allowed to collect a set of test classes, run them and retrieve the results:

```java
final Stopwatch timer = createStarted();
final Result run = new JUnitCore().run(new Computer(), TEST_CLASSES_TO_RUN);
timer.stop();
if (!run.wasSuccessful()) {
    LOG.warn("Ran [{}] test(s) in [{}] with [{}] failure(s):", run.getRunCount(), timer, run.getFailureCount());
    run.getFailures().forEach((Failure f) -> {
    LOG.warn("{}: {}", f.getTestHeader(), f.getMessage());
    LOG.debug("Caused by: {}", f.getTrace());
    });
} else {
    LOG.info("Ran [{}] tests successfully in [{}]", run.getRunCount(), timer);
}
exit(run.wasSuccessful() ? 0 : 1);
```

JUnit5 introduces the Platform Launcher API which aims to make this interface more powerful and stable. A launcher can be used to discover, filter, and execute tests. The Platform Launcher API is contained in the `junit-platform-launcher` module. If you have to run tests in an environment where you can't depend on maven-surefire like a CI environment with only a JRE installed, you can use that API to run your tests. In the following example, we have a Spring boot app that is packaged as an executable `.jar`, running a set of test classes:

```java
@SpringBootApplication
@ExtendWith(StageMatchesCondition.class)
public class Application extends GenericRunner {
   
    public static void main(String[] args) throws IOException {
        final Config config = new SpringApplicationBuilder(Application.class).web(false).build().run(args).getBean(Config.class);
        run(TestSet.fromString(config.profile()).getRequest());
    }
}
```

One way to organize your classes could be an enum like the following:

```java
public enum TestSet {
    FOO_BAR_QA("foo-bar-qa", FooIT.class, BarIT.class);
   
    [...]

    public static TestSet fromString(String name) {
        return Stream.of(TestSet.values())
            .filter(s -> s.getName().equalsIgnoreCase(name))  
            .findFirst().orElseThrow(InitException::new);

    public LauncherDiscoveryRequest getRequest() {
        final LauncherDiscoveryRequestBuilder builder = LauncherDiscoveryRequestBuilder.request();
        classesToRun.forEach(clazz -> builder.selectors(selectClass(clazz)));
        return builder.build();
    }
}
```

Your runner class would take any instance of `LauncherDiscoverRequest`, execute it and report the results:

```java
abstract class GenericRunner {
   
    static void run(LauncherDiscoveryRequest request) {
        final Launcher launcher = LauncherFactory.create();
        final SummaryGeneratingListener summaryGeneratingListener = new SummaryGeneratingListener();
        launcher.registerTestExecutionListeners(summaryGeneratingListener);
        launcher.execute(request);
        final TestExecutionSummary summary = summaryGeneratingListener.getSummary();
        summary.printTo(new PrintWriter(System.out));  
        summary.getFailures().forEach(failure -> System.out.println(String.format("%s: %s", failure.getTestIdentifier().getUniqueId(), failure.getException())));
    }
}
```

Test classes can be collected by searching for classes, package names or using all classes in the classpath. You may include filters:

```java
LauncherDiscoveryRequest request = LauncherDiscoveryRequestBuilder.request()
.selectors(
    selectPackage("com.example.mytests")
)
.filters(includeClassNamePatterns(".*Long*"))
.build();
TestPlan plan = LauncherFactory.create().discover(request);
```

The registered `SummaryGeneratingListener` generates a formatted summary of the test execution:

```
Test run finished after 235752 ms
[ 28 containers found ]
[ 0 containers skipped ]
[ 28 containers started ]
[ 5 containers aborted ]
[ 22 containers successful ]
[ 1 containers failed ]
[ 103 tests found ]
[ 1 tests skipped ]
[ 77 tests started ]
[ 2 tests aborted ]
[ 66 tests successful ]
[ 9 tests failed ]
```
It's possible to plug in custom listeners (for example to create an `.xml` report in the "Surefire" format), these need to implement the `TestExecutionListener` interface.
