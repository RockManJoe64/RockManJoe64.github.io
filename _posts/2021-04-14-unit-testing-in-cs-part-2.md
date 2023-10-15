---
layout: post
title: "Unit testing in C#: Part 2"
date: "2021-04-14"
post_image: /assets/images/blog-img-5.jpg
tags: [csharp, unit-testing]
categories: [programming]
author: jose
comments: true
dark_header: false
---

[In the last post]({% post_url 2021-04-13-unit-testing-in-cs-part-1 %}) we covered an easy template to follow to create any unit test: Arrange, Act, Assert.

In this post, we dive into practical use of libraries that will make our unit testing lives easier 😁

## Framework What?

A source of developer anxiety comes from unfamiliarity with the tools. Developers want to write unit tests, but they get entangled with the details of how to use the various frameworks available.

The following is a prescriptive list of tools you can use to start your unit testing journey. There are many more, but by sticking to these tools you'll be on your way to becoming a Testing Ninja!

## First, Some Terminology

Before we dive into the frameworks, let's go over terminology you will encounter during unit testing:

1. An **SUT** is the component whose behavior is being tested. This stands for "system under test". Most likely it is a class instance from your application you will be testing.
2. A **fixture** is a test environment that allows us to test the SUT in a controlled way. We will be going over [AutoFixture](https://github.com/AutoFixture/AutoFixture) to setup our test environment.
3. A **mock** is an object whose behavior or properties are pre-defined. In most tests, we mock external dependencies. Classes that define behavior with a datastore, REST, GraphQL, or other external data source are mocked. We will use [Moq](https://github.com/moq/moq4) for mock creation.
4. A **spy** is an extension of a mock. A spy has methods that allow us to verify how it was called by the SUT. For example, we may want to verify that an `OrderRepository`'s `CreateOrderAsync` method was called once during the test. Again, we will use [Moq](https://github.com/moq/moq4) for spies.
5. An **assertion** is the actual test. We ensure that our SUT's behavior is correct by making assertions on the output, and in some instances on called behavior of our spies. By making an assertion, you test that the actual result matches the expected result. Unit test runners already come with assertion libraries, but we will look at [Shouldly](https://github.com/shouldly/shouldly) as a way to make human-readable assertions.

Okay enough with the lecture. Onward!

### xUnit

The core framework of your unit test suite should be a unit test runner. It runs all your unit tests.

[xUnit](https://github.com/xunit/xunit) is a no-frills framework that is designed to keep the developer in the guard rails of unit testing. It defines a unit test as either one of a `Fact` or `Theory`.

A `Fact` defines a single, stand-alone unit test. It can be used if you want to test behavior with a single input.

A `Theory` defines a parameterized unit test. Use this when you want to test behavior using multiple inputs.

```csharp
[Fact]
public void ShouldAddNumbersCorrectly()
{
    // Arrange
    var x = 1;
    var y = 2;
    var expectedResult = 3;

    // Act
    var actualResult = x + y;

    // Assert
    Assert.Equal(expectedResult, actualResult);
}

[Theory]
[InlineData(0, 5)]
[InlineData(1, 4)]
[InlineData(2, 3)]
public void ShouldAddAllNumbersCorrectly(int x, int y)
{
    // Arrange
    var expectedResult = 5;

    // Act
    var actualResult = x + y;

    // Assert
    Assert.Equal(expectedResult, actualResult);
}
```

There are other test runners that can be used. They are [MSTest](https://docs.microsoft.com/en-us/dotnet/core/testing/unit-testing-with-mstest) and [NUnit](https://docs.microsoft.com/en-us/dotnet/core/testing/unit-testing-with-nunit). I'll do a comparison of all three in a future post.

### AutoFixture

[AutoFixture](https://github.com/AutoFixture/AutoFixture) is an amazing library that provides multiple benefits:

- **It provides the fixture for the test.** A fixture is a stable environment from which the test can be done. AutoFixture provides functionality to create and maintain objects needed for unit testing.
- **It is a test data factory**. We can use AutoFixture to create test data for our SUT and mocks.
- **It can integrate with Moq**. AutoFixture can be the provider of mocks for our test. This can be used when creating our actual SUT. If we use the fixture to create it, then AutoFixture will automatically inject the SUT's dependencies with mocks.

Before discovering AutoFixture, the pattern employed by me consisted of maintaining separate object factory classes to generate test data for the SUT. I also had to create mocks manually, and inject them on SUT creation. With AutoFixture, I can now eliminate all those custom object factories. I can use syntactic sugar to arrange my test properly. It makes test setup much easier.

### Moq

The [Moq](https://github.com/moq/moq4) library allows us to create mocks and spies in our unit tests. According to its Github page, it "is the only mocking library for .NET developed from scratch to take full advantage of .NET Linq expression trees and lambda expressions, which makes it the most productive, type-safe and refactoring-friendly mocking library available." Wow!

You want to use mocks when you need to define the behavior of a dependency within your class. A common usage is mocking a repository interface. If your class uses a repository to query for data, then you can mock the interface's methods to return expected objects, or throw an exception.

Use Moq when you need to create mocks, stubs, and spies for your tests. Here is an example.

```csharp
// Create the mock
var serviceMock = new Mock<IService>();

// Define mock behavior
serviceMock.Setup(m => m.SomeServiceMethod(It.IsAny<int>())).Returns(aValue);

// Define a stub behavior
var validInputId = 1;
serviceMock.Setup(m => m.SomeServiceMethod(validInputId)).Returns(expectedValue);

// Act - perform the actual test
// ...

// Verify spy behavior
serviceMock.Verify(m => m.SomeServiceMethod(It.IsAny<int>()), Times.AtLeastOnce());
serviceMock.Verify(m => m.SomeServiceMethod(validInputId), Times.Once());
```

### Shouldly

[Shouldly](https://github.com/shouldly/shouldly) takes assertions to the next level.

Not only does it make your code more readable, but the error messages that Shouldy prints makes it easy to read unit test output.

Let's substitute our `Assert` statements for `Shouldly` statements in our last example.

```csharp
[Fact]
public void ShouldAddNumbersCorrectly()
{
    // Arrange
    var x = 1;
    var y = 1; // Will cause the unit test to fail
    var expectedResult = 3;

    // Act
    var actualResult = x + y;

    // Assert
    actualResult.ShouldBe(expectedResult);
}
```

The previous `Assert` statement would print this on error:

```shell
Assert.Equal() Failure
Expected: 3
Actual:   2
```

`Shouldly` prints this error:

```shell
Shouldly.ShouldAssertException : actualResult should be 2 but was 3
```

This becomes more useful when you are doing test-driven development, or are doing more complex assertions.

## Dissection of a Unit Test

Now it's time to put it all together! Let's walk through a unit test. Consider the following example code and it's associated unit test.

**Code**

```csharp
public async Task<Order> CreateOrderAsync(Order order)
{
    _validator.ValidateAndThrow(order);

    order.Amount = order.Price * order.Quantity;

    order = await _orderTypeRepository.EnrichOrderTypeFieldAsync(order);

    order = await _orderRepository.CreateOrderAsync(order);

    return order;
}
```

**Unit Test**

```csharp
[Fact]
public async void ShouldCreateOrder()
{
    // Arrange
    // #1
    var fixture = new Fixture();
    // #2
    fixture.Customize(new AutoMoqCustomization());

    // #3
    var request = fixture.Build<Order>()
        .Without(o => o.ID)
        .Create();

    // #4
    var expectedAmount = request.Price * request.Quantity;

    // #5
    var orderTypeRepository = fixture.Freeze<Mock<IOrderTypeRepository>>();

    orderTypeRepository.Setup(m => m.EnrichOrderTypeFieldAsync(It.IsAny<Order>()))
        .ReturnsAsync((Order o) => o);

    var orderRepository = fixture.Freeze<Mock<IOrderRepository>>();

    orderRepository.Setup(m => m.CreateOrderAsync(It.IsAny<Order>()))
        .ReturnsAsync((Order o) => o);

    // #6
    var underTest = fixture.Freeze<CreateOrderService>();

    // Act
    var actualResult = await underTest.CreateOrderAsync(request);

    // Assert
    orderRepository.Verify(m => m.CreateOrderAsync(It.IsAny<Order>()),
        Times.AtLeastOnce());

    orderTypeRepository.Verify(m => m.EnrichOrderTypeFieldAsync(It.IsAny<Order>()),
        Times.AtLeastOnce());

    actualResult.Amount.ShouldBe(expectedAmount);
}
```

In the test, the behavior we are testing is a call to the `CreateOrderService`'s `CreateOrderAsync` method. It just so happens that the `OrderRepository` has a method similarly named. We are following the _Arrange/Act/Assert_ pattern to define our test.

Let's walk through the test.

### Arrange

First, we setup the test.

1. We create a fixture using the `AutoFixture` library. The library allows us to create all the objects needed for our test, as well as the SUT. Think of it as an object factory.
2. We tell the `Fixture` to use the Moq plugin `AutoMoqCustomization` in order to create mocks. Otherwise, we would call the `Moq` library directly. By doing this step, we delegate all mock creation to the fixture.
3. We use the fixture to build a request object, telling it to omit populating the ID field. The fixture will create a new object with all other properties populated with random data.
4. After creating the request object, we expect what the calculated amount will be. We will use this to assert at the end of the test.
5. Now we setup the mocks for the `CreateOrderService`'s dependencies. By using the `Freeze` method, we ask the fixture to give us singleton instances of the mocks. In this way, any further use of types `IOrderTypeRepository` and `IOrderRepository` will use the same instance internally. For each mock, we describe what the behavior of the calling methods should be. Looking at the example code, we call two methods on two dependencies: `EnrichOrderTypeFieldAsync` and `CreateOrderAsync`. The behavior is to simply return the same order object argument that is passed in.
6. Finally, we ask the fixture to give us an instance of our SUT, which is of type `CreateOrderService`. Under the hood, it will inject all required dependencies, in this case being the mocked singleton instances of `IOrderTypeRepository` and `IOrderRepository`.

### Act

Here, we call the method we are attempting to test: `CreateOrderAsync`. The method does several things:

1. It will validate the input `Order` object. The fixture will inject an `OrderValidator` that will actually validate according to set rules.
2. It will calculate the amount and set it to the `Order` object's `Amount` field.
3. It will call `IOrderTypeRepository`'s `EnrichOrderTypeFieldAsync`. We mocked this to just return the passed in argument.
4. It will call `IOrderRepository`'s `CreateOrderAsync`. Again, we mocked this to just return the passed in argument.

### Assert

Finally we make three assertions, using Shouldly's awesome extension methods:

1. We verify that `IOrderTypeRepository`'s `EnrichOrderTypeFieldAsync` method was called at least once. This ensures that the `Order` object's `OrderType` field is being set.
1. We verify that `IOrderRepository`'s `CreateOrderAsync` method was called at least once. This ensures that the order will be persisted to the data source.
1. Finally, we verify that the amount calculation is correct.

The first two assertions rely on the mocked objects acting as spies. The last assertion is directly comparing the actual amount value on the order to the expected value.

### A Parameterized Test

If you need to test your code against a set of data, then you can use a parameterized test. In such a test, you define inputs to your test, and a source of test data. Here we can take advantage of `xUnit`'s functionality.

The following example shows a test against an `OrderValidator` class. In each execution of the test, we define an `Order` object as input, but with a different property set as `null`. We expect the validator to fail due to the null properties.

The test will run a total of 4 times, once for each `Order` object created in the `GetOrders` function.

```csharp
[Theory]
[MemberData(nameof(GetOrders))]
public void ShouldFailWhenPropertyIsNull(Order order)
{
    // Arrange
    var fixture = new Fixture();
    fixture.Inject(order);

    var request = fixture.Freeze<Order>();

    var underTest = fixture.Freeze<OrderValidator>();

    // Act
    var actual = underTest.Validate(request);

    // Assert
    actual.IsValid.ShouldBeFalse();
}

public static IEnumerable<object[]> GetOrders()
{
    var fixture = new Fixture();

    yield return new object[] { fixture.Build<Order>().Without(o => o.OrderId).Create() };
    yield return new object[] { fixture.Build<Order>().Without(o => o.Price).Create() };
    yield return new object[] { fixture.Build<Order>().Without(o => o.Quantity).Create() };
    yield return new object[] { fixture.Build<Order>().Without(o => o.Symbol).Create() };
}
```

## Conclusion

In this article, we covered a lot of terminology, concepts, and example code. With these practicals in hand, I hope this opens the door to a better unit testing experience for you!
