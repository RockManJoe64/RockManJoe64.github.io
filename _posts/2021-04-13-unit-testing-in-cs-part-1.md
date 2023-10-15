---
layout: post
title: "Unit testing in C#: Part 1"
date: "2021-04-13"
post_image: /assets/images/blog-img-4.jpg
tags: [csharp, unit-testing]
categories: [programming]
author: jose
comments: true
dark_header: false
---

<!-- prettier-ignore-start -->
> "If you don’t like unit testing your product, your customers won’t like to
> test it either."<br/>
> <small>Anonymous</small>
<!-- prettier-ignore-end -->

Most of my professional experience has been in the Java language. I've written
Java code for about 12 years. In the latter part of that time, I finally
learned the joys of unit testing.

Since joining a consulting firm about 3 years ago, all of my projects have
been touching the C# language. I had to pivot on the frameworks used in C#
unit testing. It's been quite a ride, but surprisingly enjoyable!

This is meant as a primer on unit testing in C#. I want to share the
experiences I've had along the way. I've had talks in the past with other
developers concerning unit testing. It's still a gray area in our industry.

## I don't see the point

One major argument I've heard is that unit testing
**doesn't have a point.** Some developers are comfortable with
merely debugging their code to justify logic or track down bugs.

What does this say about software quality? Does this mean developers who do
not write tests do not care? Not necessarily.

In a [2019 Developer Survey](https://www.diffblue.com/Education/research_papers/2019-diffblue-developer-survey/), 83% of the software developers who were surveyed agree that their company's
software quality could be improved. In the same study, 39% said poor software
quality was due to insufficient software testing. Also in the findings,
developers agreed that "unit tests improve software quality (90% agree) and
speed up code maintenance (95% agree), making maintenance 40% faster on
average."

**Clearly developers do care about quality!** But what keeps
developers from writing unit tests? One major reason I have heard from time to
time is "I don't know how".

## I don't have the time

Another argument against unit testing:
**it's time consuming.** You're right! Writing a unit test does
take a considerable amount of time.

In [the same developer survey](https://www.diffblue.com/Education/research_papers/2019-diffblue-developer-survey/), the findings showed that developers spent on average 20% of their time
writing unit tests. An additional 15% of their time is spent writing other
forms of tests. If you take into consideration that 42% of developers have
skipped writing unit tests, then it shows developers were not spending enough
time writing these tests. The survey respondents list the reason for skipping
writing tests as there are other demands of their time, primarily feature
development.

I can't argue much against this. In my personal experience, I would spend
about 50% of my time writing tests. Writing tests consumes a great amount of
time.

However, the value in writing unit tests should validate their time demand.
The value in unit testing comes from:

- knowing that the code you wrote is bug-free,
- validates the solution to the problem you're trying to solve,
- ensuring any regressions in the code will be caught by your tests.

Developers like solving puzzles. It's satisfying to write elegant code to
solve a problem. At first, writing a unit test can be like washing the dishes;
not fun!

But writing unit tests can give you the same satisfaction as writing the code
itself. Seeing your unit tests pass in the test runner, and seeing that green
color as feedback, does give a certain level of satisfaction.

## I don't know how to unit test

Now that we're done with the appetizer, let's dig into le plat de résistance!

First, the basics.

A unit test is a single unit of code that does the following

- We first arrange the unit test. The test sets up all the test data needed, as well as the system under test. The system under test is the object you are testing. The object may be initialized with mocked dependencies at this point. We may also state certain assumptions that we expect to occur when testing.

- We then act upon the system under test. In this step, we call the specific method we are testing. The method may take as input the data we arranged at the beginning of the test.

- Finally, we assert the results of the system under test. In this step, we check that the desired output was given by the system under test. We can also verify assumptions we made when we arranged the unit test.

## An Example

This is all very abstract. To make things clear, here is an example unit test.
We can imagine we are writing a simple trade execution platform that deals
with stock market orders.

<!-- prettier-ignore-start -->
{% highlight cs %}
namespace TraderNext.Order
{
  public class OrderServiceTests
  {
    [Fact]
    public void When_CalculatingAmount_Then_ShouldBeCorrect()
    {
      // Arrange
      var quantity = 100;
      var price = 10.0;
      var underTest = new OrderService();

      // Act
      var actualAmount = underTest.CalculateAmount(quantity, price);

      // Assert
      Assert.That(actualAmount == 1000, "We expected the amount to equal 1000.");
    }
  }
}
{% endhighlight %}
<!-- prettier-ignore-end -->

What are we doing in this test? We want to test the method
`CalculateAmount` on the class `OrderService`.

### Arrange

First we setup the data. Our inputs will be the quantity and price of the
stock. We also create an instance of the OrderService as the system under
test.

### Act

We call the method under test, CalculateAmount. We give the inputs to the
method, and we name the output as actualAmount.

### Assert

We now want to check that the method returned the amount we expected.

- We use the static method Assert.That to test our condition.
- If the actualAmount equals 1000, then the test passes.
- If not, then the test fails by displaying the given error message, "We expected the amount to equal 1000".

You can use this template for all your unit tests. It's encouraged, actually,
to even include the three comments of Arrange, Act, and Assert, so that the
test is easier to read.

## To Be Continued

I've covered three reasons why developers are anxious about writing unit
tests. Developers do not see the point in writing tests, they feel they do not
have enough time, or they simply may not know how to unit test.

The only way to see the real benefit of writing unit tests is to actually
implement them. Code with missing tests is a form of tech debt, and creeping
tech debt can hinder feature development in the long run.

In the next post, I'll cover more details about unit testing, and how
to use specific frameworks to make writing tests easier. I'll also cover a
methodology for knowing what tests to actually write.

Stay tuned!
