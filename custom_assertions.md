# Take your testing DSL to the next level with custom assertions

Assertions are an important part of a test. They usually make up the `Verify` stage of a classically structured test case or happen in the `Then` stage of a test case with Given-When-Then structure. They should be easy to write, easy to read and - should they not be met - give a comprehensive report of expectation and reality. [AssertJ][1] offers great flexibility when writing assertions for all kinds of different classes e.g. `String`, `Collection` or from v3.0.0 on even `Optional`:

```java
List<String> list = Arrays.asList("a", "b", "c");
assertThat(list).as("Every element must be contained, order is important!").containsExactly("a", "b", "c");
assertThat(list).doesNotContain("d");
assertThat(list).isSorted();

Optional<String> secondElement = Optional.of(list.get(1));
assertThat(secondElement).isPresent();
assertThat(secondElement).containsInstanceOf(String.class);
```

## Stated outcome assertions and domains models

But what if you want to make assertions on your very own classes? These custom assertions can be seen as a DSL of domain assertions. They allow the creation of [so called][2] _stated outcome assertions_:

```java
Student student = new Student("Tony Starks", PROGRAM.ELECTRICAL_ENGINEERING);
assertThat(student).as("Should be enrolled in Electrical Engineering").isEnrolledIn(PROGRAM.ELECTRICAL_ENGINEERING).hasName("Tony Starks");
```

Fortunately, AssertJ allows you to create those custom assertion classes for your domain model.

Consider the following simple [example][3] class (Please find links to all code examples in the References section):

```java
public final class Student {
    private String name;
    private PROGRAM program;

    public Student(String name, PROGRAM program) { this.name = name; this.program = program; }

    public PROGRAM getProgram() { return program; }

    public String getName() { return name; }
}
```

All you have to do to is extend AssertJ's `org.assertj.core.api.AbstractAssert` in your custom assertion class and create the appropriate constructor:

```java
public class final StudentAssert extends AbstractAssert<StudentAssert, Student> {
    public StudentAssert(Student actual) {
        super(actual, StudentAssert.class);
    }
...
}
```

Entry points into these assertions classes can be placed into your own `Assertions` class or the extending class (StudentAssert in this case):

```java
public static StudentAssert assertThat(Student actual) {
    return new StudentAssert(actual);
}
```

Create expressive methods directly in your custom assertion class:

```java
public StudentAssert hasName(String name) {
    Assertions.assertThat(actual.getName())
            .as("Name must be equal to [%s]: %s", name, getDescription().orElse(EMPTY_DESCRIPTION))
            .isEqualTo(name);
    return this;
}

public StudentAssert isEnrolledIn(PROGRAM program) {
    Assertions.assertThat(actual.getProgram())
            .as("Program must be equal to [%s]: %s", program, getDescription().orElse(EMPTY_DESCRIPTION))
            .isEqualTo(program);
    return this;
}
```

Enjoy not only your new assertions methods, but everything else that already comes in AbstractAssert, like `isEqualTo`, `isNotNull`, `isInstanceOf` and so on.

As a neat little trick, the following method allows to capture the (optional) description from the test code:

```java
private Optional<String> getDescription() {
    return ofNullable(getWritableAssertionInfo().descriptionText());
}
```

In case of a failed test, these are going to be part of the error message:

```java
@Test
public void canExpectDetailedErrorMessage() {
    assertThat(student)
            .as("This is the optional description")
            .isEnrolledIn(PROGRAM.COMPUTER_SCIENCE);
}
```
```
canExpectDetailedErrorMessage(com.unitedinternet.mam.qa.StudentTest)  Time elapsed: 0.014 sec  <<< FAILURE!
org.junit.ComparisonFailure: [Program must be equal to [COMPUTER_SCIENCE]: This is the optional description] expected:<[COMPUTER_SCIENCE]> but was:<[ELECTRICAL_ENGINEERING]>
...
```

## Advanced usage

Placing the entry method directly in your domain class allows the creation of a fluent and expressive testing DSL like [the following][4]:

```java
given(PGP_SERVICE.accounts().$("PW-fBpQR7FGQEal4L356oy2rA").pgp().owned().publickeys().latest())
.when().GET().accepting(APPLICATION_PGPKEYS)
.then().assertThat()
    .statusIs(SC_OK)
        .and().containsHeader(ETAG)
        .and().as("ETag must be all numbers").headerMatches(ETAG, compile("\\d*"))
        .and().containsPgpKey()
    .and()
        .assertThatPublicKey().containsUids(PGP_ENABLED_UIDS);
```

In this advanced example taken from our PGP test project, the `then()` method returns a reponse object (wrapping RestResponse) on which some assertions should be performed. The (parent) class of this reponse object contains the entry point into the custom assertion class:

```java
public abstract class PgpResponse {
    private final RestResponse res;

    public PgpResponse(RestResponse res) { this.res = res; }
    
    public PgpResponseAssert assertThat() {
        return new PgpResponseAssert(this);
    }
    ...
}
```

The `PgpResponseAssert` class also contains a method `assertThatPublicKey`, which allows further operations on the SUT and branching out into another set of assertions:

```java
public PGPPublicKeyAssert assertThatPublicKey() {
    return new PGPPublicKeyAssert(getActual().asKeyRing().getPublicKey());
}
```

```java
public class PGPPublicKeyAssert extends AbstractAssert<PGPPublicKeyAssert, PGPPublicKey> implements And<PGPPublicKeyAssert> {
    public PGPPublicKeyAssert containsUids(Collection<Uid> uids) {
        Assertions.assertThat((Iterator<String>) actual.getUserIDs()).containsAll(convertUids(uids));
        return this;
    }
...
```

Please let me know if you have any questions about my code examples or test projects.

## References

1. http://joel-costigliola.github.io/assertj/index.html
2. http://xunitpatterns.com/XUnit%20Basics.html
3. https://git.mamdev.server.lan/qa/student-example
4. https://git.mamdev.server.lan/qa/pgp-it

[1]: http://joel-costigliola.github.io/assertj/index.html
[2]: http://xunitpatterns.com/XUnit%20Basics.html
[3]: https://git.mamdev.server.lan/qa/student-example
[4]: https://git.mamdev.server.lan/qa/pgp-it

