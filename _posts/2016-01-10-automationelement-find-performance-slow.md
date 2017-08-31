---
layout: post
title: "AutomationElement Find Performance Slow"
date: 2016-01-10
tag: automation
---

When working with the Microsoft UI Automation framework on Windows 10 I came across an interesting problem.  The API we were developing on top of the MS UI Automation framework had since been able to find all our requested elements in a timely manner back when we were developing and testing on Windows 8.1.

However, after upgrading to 10 and .NET 4.6 performance on finding these elements tanked.  There were no code changes we had made to our layer, but we could replicate the performance issues on Win 10 on demand.

The specs of both boxes were exactly the same at:

 - Xeon W3670 3.2GHz CPU
 - 12gb RAM
 - 64bit OS

Here's my performance test results:

WPF Default DataGrid, no styling, triggers or any other fancy stuff and issuing a UI Automation call to AutomationElement.FindAll() to get all the DataGrid cells in each row of the DataGrid:

Win 8.1/ .NET 4.5.1:

- 5-10ms to get all visible DataGrid cells on average.

Win 10 / .NET 4.6:

- 30-50ms to get all visible DataGrid cells on average.

In case you're wondering the DataGrid under test had Virtualization on (set to Recycling).

Having also tried using the unmanaged MS UIA API via COM Wrapping and seeing no change in performance I opened up a support ticket.  After working with them for a bit it turns out this was an issue with:

"UIA was frequently calling the NtQuerySystemInformation, the performance of calling that API frequently is not satisfactory. They made changes to that particular code path and do not call that API anymore and that improved the overall performance." - MS Support Rep

I couldn't get any more useful information than that.  However, we installed Win10 updates and found that the issue was fixed.  More detail on the fix can be found here:

https://support.microsoft.com/en-us/kb/3093266

Hope this helps someone who is running into the same issue.