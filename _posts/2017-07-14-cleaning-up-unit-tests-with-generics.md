---
layout: post
title: "Cleaning up Unit Tests with Generics"
date: 2017-07-14
tags: testing c#
---

I recently finished Roy Osherove's The Art of Unit Testing: with examples in c# (2nd edition). Going into the book I had been writing unit tests and doing automated testing for a few years in C#. I was eager to see if I had some bad habits as well as fill in gaps in my understanding of the fundamentals. Osherove assures his readers that the book contains content geared towards beginners as well as devs with unit testing experience. Osherove quickly gets into the meat of unit testing and I learned some valuable techniques I've already begun using and wanted to share.

Part 3 of the book (titled 'The Test Code') was what I was most interested in and where I believed I had the most gaps in terms of practical knowledge. In this part he covers how to organize your tests and commonalities of good unit tests and unit test hierarchies. This section is broken up into two good-sized chapters, covering not only writing and organizing tests, but how to properly run them so their usefulness is maximized.

I specifically liked the section on using inheritance with test classes to minimize test duplication across similar classes. This was something I had tried with NUnit before, but ended up not keeping. In the past I had used generics with TestFixtures by having parameters in the TestFixture attribute contain types that I wanted to use as class level constraints so I could reuse the tests. Here's what that looked like:

```csharp
[TestFixture (typeof (IMove), typeof (DefaultMoveService))]
[TestFixture (typeof (IMovement), typeof (DefaultMovementService))]
[TestFixture (typeof (ICharacter), typeof (DefaultCharacterService))]
[TestFixture (typeof (ICharacterAttributeRow), typeof (DefaultCharacterAttributeService))]
public class GeneralServiceTests<TModel, TSut>;
where TModel : IModel
where TSut : ICrudService<TModel>; {
    //Tests...
    //method to create instances of the types used in the
    //class test cases
    private static ICrudService<TModel>;
    CreateCrudServiceSut (IRepository<TModel> repository) {
        return (ICrudService<TModel>) Activator.CreateInstance (typeof (TSut), repository);
    }
}
```

This was really clunky and I found the Resharper Unit Test Explorer in Visual Studio would often have trouble showing them and I'd be faffing around with it for far too long just trying to get it show all my tests. Aside from that, I found it painful to make changes to these tests since a large portion of what is being tested lie outside of the class and outside of the actual test methods. In order to not break tests I had to make sure all these types had the same constructor requirements and method signatures. Definitely not ideal.

Osherove shows examples using an abstract base test class and abstract factory (necessary test fakes) methods with inheritance to create a much more natural hierarchy than what I initially had. Instead of using NUnit's TestCase attribute, I could make this base test class abstract and then make my factory methods abstract and return the generic type specified on the class. This helped with my above example where I had no good way of modifying the CreateCrudServiceSut() for the types used in the tests without causing an unknown amount of tests to break at runtime. With this approach I could be much more straightforward. Also, since each type would get its own test class, the Resharper Unit Test Explorer extension would stop complaining about multiple instances of the same test class. Now each would show up as its own test class.

In the book, Osherove calls this the 'Abstract Driver Test Class' pattern or the Abstract 'fill in the blanks' Test Driver Class pattern. More info on it can be found here (Unfortunately, it looks like the images on the page can't be retrieved anymore, but the descriptions are still present).

Now I could refactor these tests and break them out into their own class where I could implement more robust factory methods. Keep in mind, the actual tests I want to reuse are still located in the base test class. I simply override the types used in the derived test classes. This way I do not have to rewrite the same tests and later I can add tests specific to the derived classes without mucking up the base class.

Now my example test class looks like this: (I'm just showing a single derived test class)

```csharp
[TestFixture]
public class CharacterServiceTests : GeneralServiceTests<ICharacter> {
    protected override ICrudService < ICharacter CreateCrudServiceSut (IRepository<ICharacter> repository) {
        return new DefaultCharacterService (repository);
    }

    //Tests inherited from GeneralServiceTests class
}

public abstract class GeneralServiceTests<T>;
where T : IModel {
    //Tests...

    //method to create instances of the types used in the
    //class test cases
    protected abstract ICrudService<T> CreateCrudServiceSut (IRepository<T> repository);
}
```

This is just one of the many concepts taught in Osherove's book that really helped me. I highly recommend checking it out if you're looking to improve on your unit test designs and fundamentals.

[The Art of Unit Testing: with examples in C# - Roy Osherove - 2nd Edition](https://www.amazon.com/Art-Unit-Testing-examples/dp/1617290890/)
