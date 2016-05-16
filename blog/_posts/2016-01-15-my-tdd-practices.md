---
layout: post
author: Matt Dickinson
title:  "My TDD practices"
date:   2016-01-15
tags:
  - tdd
---

This blog post is an adaptation from a presentation I frequently gave to coworkers. I am a big proponent of [Test Driven Development](https://en.wikipedia.org/wiki/Test-driven_development)
(TDD) and use it both professionally and on personal projects.

**Note**: I understand that TDD is not a magic bullet. For simplicity's sake, I'm using absolutes rather than saying "TDD *may*" with everything. 

## Why TDD?

First and foremost, **TDD improves code quality**. If it didn't, I can't imagine anyone pushing to do it.

Secondly, **TDD improves architecture quality**. Bad architecture is hard to unit test, so if the developer is driven enough to do proper TDD,
their refactoring will result in an improved structure. When writing unit tests, separation of concerns is very apparent. The developer should
ask themselves whether faking a database call for a business logic test is needed.

TDD also gives an improved **sense of security around refactoring**, assuming the code had been properly unit tested. As long as the interfaces remain the same,
the guts of the code can be changed as needed and the unit tests will still pass.

There's also the pure math of it all. There's a video I really enjoyed called [Integrated Tests Are A Scam](https://vimeo.com/80533536) that really breaks this down.
Let's say we have a web service call with three tiers: controller, business logic, and database access. The controller has two logic paths, the business has seven, 
and the database access has three.
If one were to fully test all logic paths through the three layers via integration testing, it would require `2 * 7 * 3 = 42` test cases.
However, fully unit testing the logic paths would require `2 + 7 + 3 = 12` test cases. The difference really adds up as more features are added.
Of course, unit testing does not entirely exclude one from integration testing, but it doesn't require nearly as much.

**Touching old (unit tested code) is less scary**. Whether it's someone else's code or your own code from six months ago (which is basically someone else's code),
the risk of introducing bugs is greatly decreased if it's backed by unit tests.

## What doesn't TDD do?

As I mentioned before, **unit testing does not replace integration testing**. For one thing, integration testing is needed to make sure the contracts
between layers are being respected. For example, the business layer might be coded assuming that the data access layer will return null if a record doesn't exist,
whereas the data access layer will actually throw an exception. Testing each layer by itself will not highlight this issue.

**There is such a thing as a bad unit test**. I had a coworker who was tasked with going back and adding unit tests for the whole project (which I strongly discourage).
He did it, but the tests really only tested the happy path and sometimes didn't even run any assertions. At best, they only verified that the code didn't throw any unhandled exceptions.

**Incomplete test coverage can also give a false sense of security**. For teams doing code reviews (which I strongly encourage), unit tests should be included as part of that code review.
This could help identify missing test cases.

Similarly, **bad code can satisfy the best unit tests**. You can still have a well-defined interface backed by a pile of spaghetti code.

**Not all code can be unit tested**. For example, if the code needs to interact with a file on the filesystem, this should not be included in the unit test.
The general guideline is to create a class that encapsulates the "stateful" aspects and then write your code against this interface. The goal is to get as close to the boundaries
of the system as possible and then use integration testing for the boundary classes.

Finally, **unit testing does not guarantee that your requirements are correct**. You could fully unit and integration test your code, but if the requirements don't fit business needs you have a bigger problem.

## Core Principles

* No new production code can be written without a failing unit test. Non-compiling code is considered a failing unit test.
* New production code should only be written to satisfy a failing unit test.
* Only one class and method should be tested at a time
* Small tests are preferable
* Tests are organized into Arrange, Act, and Assert

**Arrange** – Data and stub setup. What data are we pretending exists in the database? What HTTP response code should this fake service return? What's the request I'm going to be passing in to the code being tested?

**Act** – Call the method being tested. This is usually a single line

**Assert** – Verify that the code performed as expected. Did I get the right values back? Did I correctly call the downstream code?

## Good TDD Practices

### Each test should be fairly self-describing

The common practice is `MethodBeingTested_preconditionsOfTest_expectedOutcomeOfTest`. This is compatible with the Business Driven Development (BDD) "Given, When, Then" structure,
albeit in a slightly different order. For example, a secured system could have a test like `ViewDashboard_authorizationIsSetWithUnrecognizedType_returns401`

### Data setup should be contained within each test

This goes against a lot of development conventions, like Don't Repeat Yourself (DRY). I've worked in test suites that have a common data setup step and it always turns into an issue.
First and foremost, it reduces code readability. When you're 300 lines down in a test suite, you don't want to have to scroll all the way back to the top to see what properties the "TestUser3" user has.
Following in a close second, a common data setup makes the tests very brittle. You've now made tests dependent on each other. Changing one thing for one test could now adversely affect every other test.
However, DRY principles still have a place in unit tests. I usually take the approach of adding helper methods for generating test data, like `GetUser(string username, bool isVerified, string emailAddress)`.
Then, if a test needs more specific setup, it can modify the object returned by the helper method. This results in a harmony between reusability and readability.

### Every test case should check input and/or output

Otherwise, the only thing the test accomplishes is verifies that no unhandled exception was thrown. Similarly, every input and output case should have at least one test.
Note that I said input and output case. If you allow a username between 5 and 20 characters, you don't need to generate all possible lengths. You should check:

* Null/empty username results in failure
* Four characters results in failure
* Five characters results in success
* Twenty characters results in success
* Twenty-one characters results in failure

### Make it easy on yourself

**The best tool I have for unit testing is ReSharper**. I frequently take advantage of the templating functionality. I have a [file template](https://gist.github.com/mdickin/e46095a3100b4d38999acd747a78a9a2)
for the scaffolding of a unit test file, as well as a [shortcut to insert a new unit test](https://gist.github.com/mdickin/4b7400b88e5d8d66cfccceaf9beac3e4#file-testm_live_template-dotsettings). 
I type `testm`, hit tab a few times, and it generates the following:

```csharp
[TestMethod]
public void MethodName()
{
    //Arrange
    
    
    //Act
    
    
    //Assert
    Assert.Fail();
}
```

From here, I type the method name for the case I'm testing in the aforementioned format and go on. A lot of the time, I'll generate many empty unit tests at a time while I'm thinking through the different test cases.
The `Assert.Fail()` at the end helps me in the event that I get interrupted and have to come back to whatever I was doing. Without it, the test would succeed and give me the false impression that the work was done.

As I touched on before, **I make frequent use of helper methods** for data setups or complex assertions. I also use helper classes for the more cookie cutter structures like SQL repositories or HTTP clients.

**Test values can be a pain point if you let them**. I try to use simple values that can't be a mistake. For example, rather than setting an Id to 1, I'll set it to 23456.
Similarly, unless it's pertinent to the test case, I try not to put too much effort into values without impacting readability. 
Instead of setting an address to `{ "1600 Pennsylvania Ave", "Washington DC" }`, I'll use `{ "address1", "cityName" }`. 
I would also recommend testing against those literal values in your Assert section rather than referencing a previously set variable.

Bad practice - "interesting" values and reference values in assertions

```csharp
[TestMethod]
public void MethodName()
{
    //Arrange
    var address = new Address
    {
        Address1 = "1600 Pennsylvania Ave",
        City = "Washington DC"
    };
    _addressRepository.Setup(mock => mock.Get(It.IsAny<int>()))
        .Returns(address);
        
    //Act
    var result = _business.GetAddress(123);
    
    //Assert
    Assert.AreEqual(address.Address1, result.address1);
    Assert.AreEqual(address.City, result.City);
}
```

Good practice – "boring" values and literal assertions

```csharp
[TestMethod]
public void MethodName()
{
    //Arrange
    var address = new Address
    {
        Address1 = "address 1",
        City = "city"
    };
    _addressRepository.Setup(mock => mock.Get(It.IsAny<int>()))
        .Returns(address);
        
    //Act
    var result = _business.GetAddress(123);
    
    //Assert
    Assert.AreEqual("address 1", result.address1);
    Assert.AreEqual("city", result.City);
}
```

**If you're testing whether you find something in a list**, never put that item as the first (or only) item in the list. You don't want a false positive when your code is actually just calling `collection.First()`.

**Start from the top**, meaning the entry point into your system. In ASP.NET MVC or Web API, this would be the Controller. This is the best way to generate the minimum viable product (MVP).
I've done it the other way before, and I ended up generating a lot of code that wasn't actually needed to accomplish the feature. If you start with the data access layer,
it's tempting to say "Well, we'll need all the CRUD operations here" when in fact your story only requires a Read action.

## Pitfalls

**TDD can be painful to switch gears to**. It seems counter-intuitive at first, and it's not obvious that the extra work up front reduces work down the road.
Even if you're already comfortable with TDD, your teammates may not be. It can be a constant uphill battle convincing new people of the advantages.

**Testing will occasionally get in your way**. Sometimes you can spend a lot of time creating classes to wrap the system boundaries. Also, some 3rd party libraries aren't written with unit tests in mind,
so you may have to put in extra time to write an abstraction layer.

**TDD might require architectural changes for existing code**. If your codebase does not use a [Dependency Injection](https://en.wikipedia.org/wiki/Dependency_injection) (DI) framework
or if you have all your logic in a single class, you might have a significant amount of refactoring on your hands. However, the time is completely worth it, even if you don't end up doing TDD.

## Summary

Even given the difficulties with TDD, it is still completely worth it. Eventually, you get to a point where writing code that isn't backed by a unit test will make you feel uncomfortable. The extra time spent writing the
unit test is more than recovered by the decreased time fixing bugs or refactoring down the road.