---
layout: post
title: "MSTest v2 - New (old) kid on the block"
date: 2017-08-18
tags: testing c#
---
It's been a long time since MSTest was in discussions of modern testing frameworks.  However, here we are in 2017 and MSTest is getting active updates once again, due in part to it being open-sourced.  Let's take a look at what some of these new and exciting features are.

Please note, this post will cover features that are currently in `pre-release` for the MSTest v2 NuGet package.  

**NOTE**: Remember to use the `pre-release` flag with the NuGet package you will be using:
- [MSTest.Framework](https://www.nuget.org/packages/MSTest.TestFramework/1.2.0-beta)
- [MSTest.TestAdapter](https://www.nuget.org/packages/MSTest.TestAdapter/1.2.0-beta)

Visual Studio 2015+ is also recommended; your mileage may vary with 2013.

This beta release is NOT the first public iteration of MSTest v2, but contains some of the features discussed below.  Having stronger support around parameterized test cases is very important and those features are in the 1.2 beta; version 1.1.13 was the first v2 [release from github](https://github.com/Microsoft/testfx-docs/blob/master/docs/releases.md#1113).

Release notes for each version can be found [here](https://github.com/Microsoft/testfx/releases).

On a post about the future of MSTest v2 and mstest.exe, user (and presumed employee of Microsoft) 'Abhitej_MSFT' clarifies that:

>>MSTest V2 tests are not supported with “mstest.exe”. In the TFS build template the Test Runner should be “Visual Studio Test Runner”(https://msdn.microsoft.com/en-us/library/ms253138(v=vs.110).aspx#Runner) . I hope your definition does not require the legacy testsettings. Do let us know on aajohn@microsoft.com if you hit any issues.

Source - [https://blogs.msdn.microsoft.com/devops/2017/02/25/mstest-v2-now-and-ahead/#comment-78796](https://blogs.msdn.microsoft.com/devops/2017/02/25/mstest-v2-now-and-ahead/#comment-78796)

I have yet to come across an MSTest v2 feature that NUnit has not already had for quite some time.  However, the features discussed here are still very useful if you decide to use MSTest over another test framework.

For those who want to see it all, the repository for MSTest can be found on [github](https://github.com/Microsoft/testfx).

## Features


### DynamicData [#141](https://github.com/Microsoft/testfx/issues/141)

Parameterized testing has long been available in MSTest, using the `DataRowAttribute` which look like this:

```csharp
[TestMethod]
[DataRow (1, 2, 3)]
[DataRow (4, 5, 6)]
public void MyParameterizedTest (int a, int b, int c) {
    //perform assertions on a,b and c
}
```

The resulting two tests each use one instance of the 'rows' of data.  However, what if we want to reuse these same values across tests?  As it would be inefficient to have to duplicate these values on every test where they are used, v2 offers `DynamicDataAttribute`:

```csharp
private static IEnumerable<object[]> ReusableTestData =>
    new List<object[]> {
        new object[] { 1, 2, 3 },
        new object[] { 4, 5, 6 }
    };

[TestMethod]
[DynamicData (nameof (ReusableTestData))]
public void MyParameterizedTest (int a, int b, int c) {
    //perform the same assertions the same as before.
}
```

Now this test data can be used on any number of tests without having to duplicate the actual data, making the process cleaner.  Those of you who are familiar with NUnit may know this as the `TestCaseSourceAttribute`.

Methods can also be used as test data. To do so, simply use the overload of the `DynamicDataAttribute` constructor that takes in a `DynamicDataSourceType.Method` `enum`.  By default, the framework will assume the name of the dynamic data passed in is a `Property`.

***

### Custom Test Data for Parameterized Tests [#141](https://github.com/Microsoft/testfx/issues/141)

Taking `DynamicData` one step further is useful if the parameters used in your tests are a bit more complex.  Using the `CustomTestDataSourceAttribute` you can now create an attribute that loads up this data for you to consume in your tests while keeping your test nice and clean.  Here is the end result:

```csharp
[TestMethod]
[CarTestData]
public void ATestUsingACar (string model, int year) {
    var carUnderTest = new Car { Model = model, Year = year };
    //perform assertion on the car object.
}

//config for the 'CarTestData' attribute:
class CarTestDataAttribute : Attribute, ITestDataSource {
    public IEnumerable<object[]> GetData (MethodInfo methodInfo) {
        return new List<object[]> {
            new object[] { "Ford", 1990 },
            new object[] { "Nisson", 2017 }
        };
    }

    public string GetDisplayName (MethodInfo methodInfo, object[] data) {
        if (data != null) {
            return string.Format (CultureInfo.CurrentCulture, "{0} ({1})", methodInfo.Name, string.Join (",", data));
        }
        return null;
    }
}

public class Car {
    public string Model { get; set; }
    public int Year { get; set; }
}
```

Creating a new `Attribute` that implements the MSTest v2 `ITestDataSource` interface allows us to set up our test objects outside of the test, provide a variable amount of them for parameterized tests, and reuse them in other tests without duplication.

This can be pretty powerful in some scenarios, however I am not entirely in love with `ITestDataSource.GetData()` returning an `IEnumerable<object[]>` instead of returning an `IEnumerable<object>`, `IEnumerable<T>` or offering some other interface.  This makes it somewhat cumbersome to use this mechanism to return non-primitives. 

`ITestDataSource.GetDisplayName` will be displayed when the test is executed via the target runner (VSTest if running in Visual Studio), allowing us to determine which cases of a test failed when tests are parameterized.

***

### Assert.That [#116](https://github.com/Microsoft/testfx/issues/116)

With the introduction of a focused extension point for assertion logic, MSTest v2 now offers `That`, a static property on `Assert` which returns the instance and allows for an easy jumping point for extension assertion methods.

Here's an example:

```csharp
public static class MyAssertExtensions {
    public static void CountIsGreaterThan<T> (this Assert assert, IEnumerable<T> objs, int num) {
        int actualCount = objs.Count ();
        if (actualCount > num) {
            return;
        }
        throw new AssertFailedException ($"Expected {nameof(objs)} count to be greater than {num}, but was {actualCount}.");
    }
}
```

This means tests are much more readable in their assertions:
```csharp
[TestClass]
public class MyTests {
    [TestMethod]
    public void ATest () {
        var aList = new List<int> { 1, 2, 3, 4 };
        Assert.That.CountIsGreaterThan (aList, 0);
    }
}
```
`Assert.That` can be used to replace and/or supplement calls to `Assert.AreEqual()` and `Assert.IsTrue()`.  By themselves, these are such broad calls that often times they can lead to confusion when debugging or reading tests.  In my experience this is especially true with `Assert.IsTrue()` due to its default failure output of `Assert.IsTrue failed`. `AreEquals()` attempts to mitigate any confusion by reporting the actual and expected values.

NOTE:  If you are not going the route of extension methods a quick way to help solve this problem is to pass the name of the property/method under test when passing a failure message.  An example:

```csharp
public void MyTest () {
    var aList = new List<int> { 1, 2, 3 };

    //nameof() is a C#6 feature, but is not required to make this work.
    Assert.IsTrue (aList.Count > 3, $"{nameof(aList.Count)}");

    //failure prints: Assert.IsTrue failed. Count
}
```

And an example of enhancing `Assert.IsTrue ` with this type of approach:

```csharp
public static class MyAssertExtensions {
    public static void IsTrue<T> (this Assert assert, T instance, Expression<Func<T, bool>> assertionExpression) {
        if (assertionExpression.Compile ().Invoke (instance)) { return; }
        throw new AssertFailedException ($"Assertion failed for expression {assertionExpression}");
    }
}

//Example
public void IsTrue_Extended_Test () {
    var aList = new List<int> { 1, 2, 3, 4 };
    Assert.That.IsTrue (aList, list => list.Count > 4);

    //failure prints: Assertion failed for expression 'a => (a.Count > 4)'.
}
```

Taking this a bit further we can create a fluent syntax that lends itself to more extensibility in the future:

```csharp
[TestMethod]
public void IsGreaterThan_Redux () {
    var aList = new List<int> { 1, 2, 3, 4 };
    Assert.That.For (aList).IsTrue (list => list.Count > 4);

    //failure output: Assertion failed for expression 'list => list.Count > 4'
}

public static class MyAssertionExtensions {
    public static For<T> For<T> (this Assert assert, T instance) {
        return new For<T> (instance);
    }
}

public class For<T> {
    private readonly T _instanceUnderTest;

    public For (T instanceUnderTest) {
        _instanceUnderTest = instanceUnderTest;
    }

    public void IsTrue (Expression<Func<T, bool>> assertionExpression) {
        if (assertionExpression.Compile ().Invoke (_instanceUnderTest)) { return; }
        throw new AssertFailedException ($"Assertion failed for expression '{assertionExpression}'.");
    }
}
```
Clearly, creating extension methods with `Assert.That` heavily mitigates the readability problem.  For an additional point of reference, see the [original RFC](https://github.com/Microsoft/testfx-docs/blob/master/RFCs/002-Framework-Extensibility-Custom-Assertions.md).

While this is not a comprehensive overview of the new features in the pipeline for MSTest v2, I have covered those that strike me as being particularly useful.  

For more information on MSTest v2 check out these links:

- [Repository](https://github.com/Microsoft/testfx)
- [Documentation](https://github.com/Microsoft/testfx-docs)
- [Initial Microsoft Blog post for v2](https://blogs.msdn.microsoft.com/devops/2016/06/17/taking-the-mstest-framework-forward-with-mstest-v2/)

MSTest v2 is definitely making MSTest a more relevant testing framework.  The concepts here are the aspects I was most interested in as a current user of NUnit.
While I myself have not yet made the switch from NUnit to MSTest v2, these new features have made considering the possibility much more favorable. Hopefully these features combined with more features coming down the pipeline in the future continue to improve developers view of MSTest when compared to other testing frameworks. 