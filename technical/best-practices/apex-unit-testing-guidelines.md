# Apex Unit Testing Guidelines

## General Guidelines

- The goal of unit testing is to enable sustainable growth of the software project.
- If you find that code is hard to unit test, it's a strong sign that the code needs improvement. The poor quality usually manifests _tight coupling_.
However, the fact that you can easily unit test your code base doesn't necessarily mean it's of good quality.
- _Not all tests are created equal._ Some of them are valuable and contribute a lot to overall software quality. Others don't.
To enable sustainable project growth, you have to exclusively focus on high-quality tests.
- _Code is a liability, not an asset._ The more code you introduce, the more you extend the surface area of potential bugs. _Tests are code, too._
You should view them as part of your code base that aims at solving a particular problem: ensuring the application's correctness.
Unit tests, just like any other code, are also vulnerable to bugs and require maintenance.
- Tests shouldn't verify _units of code_. Rather, they should verify _units of behavior_:
something that is meaningful for the problem domain and, ideally, something that a business person can recognize as useful.
- Aim for black-box testing over white-box testing. Make all tests view the system as a black box and verify behavior meaning ful to the problem domain.
    - When _writing tests_ you can still use the white-box method when _analyzing_ the tests (e.g. to see which code branches are not exercised).

## Definition of Unit Test

A unit test is an automated test that
- Verifies a small piece of code (also known as _unit_).
- Does it quickly.
- And does it in an isolated manner.

An integration test is a test that doesn't meet at least one of the criteria for a unit test.

For **true unit testing** in Apex:

- **No DML operations** - Mock database interactions
- **No SOQL queries** - Use dependency injection or mocking frameworks

## Test Structure

### Arrange-Act-Assert Pattern

Always structure your tests using the **Arrange-Act-Assert** (aka Given-When-Then) pattern.

- Arrange: bring the system under test (SUT) and its dependencies to a desired state.
    - It's usually the largest of the three. But if it becomes significantly large, it's better to extract the arrangements either into private methods within the same test class or to a separate factory class (consider the Object Mother design pattern).
- Act: call methods on the SUT, pass the prepared dependencies, and capture the output value (if any).
    - It's normally just a single line of code. If it consists of two or more lines, it could indicate a problem with the SUT's public API.
- Assert: verify the outcome. The outcome may be represented by the return value, the final state of the SUT and its collaborators, or the methods the SUT called on those collaborators.
    - A single unit of behavior can exhibit multiuple outcomes, and it's fine to evaluate them all in one test.

Example:

```apex
@IsTest
private class AccountServiceTest {
    
    @IsTest
    static void shouldCreateAccountWithProperDefaults() {
        // ARRANGE
        String accountName = 'Test Account';
        AccountService service = new AccountService();
        
        // ACT
        Test.startTest();
        Account result = service.createAccount(accountName);
        Test.stopTest();
        
        // ASSERT
        Assert.areEqual(accountName, result.Name);
        Assert.areEqual('Prospect', result.Type);
    }
}
```

### Test.startTest() and Test.stopTest()

Always surround the _act_ block** with `Test.startTest()` and `Test.stopTest()`:

- Resets governor limits for the test execution
- Ensures proper isolation of the code under test
- Forces completion of asynchronous operations
- Provides accurate performance measurements


## Test Naming Guidelines

- Name the test as if you were describing the scenario to a non-programmer who is familiar with the problem domain.
- Don't include the name of the SUT's method in the test's name. Remember, you don't test _code_, you test _application behavior_.
  - The only exception to this guideline is when you work on utility code. Such code doesn't contain business logic, so it's fine to use the SUT's method names there.
