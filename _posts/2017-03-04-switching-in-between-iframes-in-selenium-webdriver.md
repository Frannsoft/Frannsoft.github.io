---
layout: post
title: "Switching in-between IFrames in Selenium Webdriver"
date: 2016-01-10
tags: automation webdriver c#
---

It would be nice if iframe support in WebDriver was a little more robust, but there are ways to help the situation. Selenium's WebDriver only searches for elements in the current loaded DOM and it considers each iframe to be a separate DOM. You'll need to manage that switching on your own as needed even after the PageFactory has been run against the Page Object in question and web elements found and utilized.

One option worth considering is creating some utility methods that handle switching in and out of iframes on the Page Object.

Here's an example using the C# WebDriver bindings. The concept can translate to bindings in other languages as well:

```csharp
public class MyPageObject {
    private readonly IWebDriver _driver;

    [FindsBy (How = How.Id, Using = "id")]
    private IWebElement _myWebElementInIFrame;

    public MyPageObject (IWebDriver webDriver) {
        PageFactory.InitElements (webDriver, this);
        _driver = webDriver;
    }

    public void DoStuff () {
        SwitchToIframe ("myiframe");
        _myWebElementInIFrame.Click (); //element found using PageFactory data now
        SwitchToDefaultContent ();
    }

    public void SwitchToIframe (string frameName) {
        _driver.SwitchTo ().Frame (frameName);
    }

    public void SwitchToDefaultContent () {
        _driver.SwitchTo ().DefaultContent ();
    }
}
```

You could take this a bit further using some method templating:

```csharp
public class MyPageObject : BasePageObject {
    [FindsBy (How = How.Id, Using = "id")]
    private IWebElement _myWebElementInIFrame;

    public MyPageObject (IWebDriver webDriver) : base (webDriver) {
        PageFactory.InitElements (webDriver, this);
    }

    public void DoStuff () {
        ExecuteActionInIFrame (() => {
            _myWebElementInIFrame.Click (); //element found using PageFactory data now
        }, "myiframe", "mynestediframe");
    }
}

public abstract class BasePageObject {
    protected IWebDriver WebDriver { get; }

    protected BasePageObject (IWebDriver webDriver) {
        WebDriver = webDriver;
    }

    public void ExecuteActionInIFrame (Action action, params string[] frameNames) {
        foreach (string frameName in frameNames) {
            WebDriver.SwitchTo ().Frame (frameName); //switch into multiple nested iframes in passed in order
        }

        //perform action now that we're in proper iframe
        action ();

        //bounce out to root to have a standard starting point for next action
        WebDriver.SwitchTo ().DefaultContent ();
    }
}
```

This base Page Object class can now handle switching into several layers of iframes to perform an action before switching back out which deriving Page Objects can take advantage of as needed.

Placing this logic in the Page Object will help the test writer avoid having to account for iframe switching in their tests. In most cases, these types of UI Automation tests focus on the business cases explicitly and most users do not care about what iframe they are in when using your product. Putting this logic in the Page Object will also help with maintenance since product changes involving these iframes can just be changed in the Page Object rather than having to update a potentially large amount of tests individually.

Another tip that helped me was one can get the name of a current iframe via javascript `window.name`. This can be returned using `IJavaScriptExecutor` which is implemented by `IWebDriver` in the dotnet bindings. This might help you determine where exactly you are in the overall DOM while traversing iframes.

Hopefully this example helps someone else coming across a similar problem when automating the UI of products that use iframes.

