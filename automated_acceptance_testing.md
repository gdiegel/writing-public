# Automated acceptance testing
> The key words MUST, MUST NOT, REQUIRED, SHALL, SHALL NOT, SHOULD, SHOULD NOT, RECOMMENDED, MAY, and OPTION AL in this document are to be interpreted as described in [RFC 2119][1].

## Introduction
An application deployed in any environment requires extensive functional testing before propagation into the next stage. Functional testing is conducted to evaluate the compliance of a system or component with specified functional requirements, it is not concerned with application internals.

Once your software has passed a set of unit and integration tests in the commit stage, a release candidate is built and it will be deployed into an environment where it will have to pass a second stage of tests. In this article we propose the concept of a test artifact. These tests described here will be carried out by this specialized artifact which will have to be built during the build step of the regular application or separately in a ("physically") unrelated project.

The scope of the tests performed by the test artifact described here is acceptance testing. Acceptance tests will assess whether an artifact meets functional requirements and is acceptable for deployment. Definition of acceptance testing by ISTQB:
> Acceptance testing: Formal testing with respect to user needs, requirements, and business processes conducted to determine whether or not a system satisfies the acceptance criteria and to enable the user, customers or other authorized entity to determine whether or not to accept the system.

Automated acceptance tests contextually belong in the upper-left quadrant of the famous agile testing quadrants, originally proposed by [Brian Marick][1]:

![Agile testing quadrants](images/agile_testing_quadrants.png)

## Acceptance criteria
Acceptance tests are created from acceptance criteria. These criteria MUST be defined before development on a story starts and acceptance tests should be derived from these criteria as early as possible. It is invaluable to involve everyone on the team as much as possible in the process of designing acceptance criteria. Only if everyone is on the same level, so to speak, are you able to benefit from shared knowledge and minimize trouble down-the-line. It is a great way to create a common understanding of the value of the new feature and to nip potential misunderstandings in the bud. One should avoid making assumptions based only on one's own perception. Communication (as in every aspect of life) is key!

## Best practices
### Ownership
Acceptance tests should be owned by everyone because they provide value for everyone, they are a collaborative effort. They should be familiar to each person in an agile development team and everyone should know their scope. Everyone shares responsibility for these tests and fixing them should always get the appropriate priority.
### Propagation
An artifact should not be deployed to production without first having passed the entire acceptance test stage by debing deployed to the first environment and propagating through all available stages towards production. It should be easier to fix the tests and pass the acceptance test stage than to circumvent the stage entirely.
### Atomicity
Acceptance tests should be atomic in that they are self-contained and can (and should!) run independently from each other. There must not be a given order to run the tests in. In addition to that, they should be able to run in parallel without creating race conditions. This means that ideally each test uses unique test data over the dimensions time and space. This implies ephemeral test data which is dynamically created and destroyed after the test suite has finished running.
### Information
Acceptance tests should provide fast and reliable feedback whenever an error occurs and should also provide the necessary information to identify the source of the error as a quickly as possible.
### Flakiness
Flaky tests (Toggling, flipping or otherwise unreliable tests) provide no value to an agile development team. Even worse, they decrease confidence in the acceptance test suite and should be eliminated promptly. Use retry mechanics in your tests if possible! If a test is still flaky, throw it into the the trash.
### Robustness
Acceptance tests should always be kept green. Run them at least nightly and fix any failures immediately.
### Duration
Acceptance tests should be reasonably fast. Use parallelization wherever possible while abvoiding race conditions.

[1]: https://tools.ietf.org/html/rfc2119
[2]: http://www.exampler.com/old-blog/2003/08/21.1.html#agile-testing-project-1