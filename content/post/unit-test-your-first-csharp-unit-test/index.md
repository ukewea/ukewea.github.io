---
title: "Unit Test Your First Csharp Unit Test"
description:
date: 2022-12-23T21:59:42+08:00
image:
math:
license:
hidden: false
comments: true
draft: false
categories:
  - Unit Testing
tags:
  - unit-testing
  - testing
  - software-testing
---

## Scenario

I have a `DemoLibrary` project written in C# and need to add a unit test project `DemoLibrary.UnitTests` to hold tests related to `DemoLibrary`.

---

## 1. Create a New Project

Within the solution, choose to add a new project.

![](2024-04-30-22-08-20.png)

Select an xUnit test project, then click 'Next'.

![](2024-04-30-22-08-28.png)

Name your test project, preferably ending with `.UnitTests` to easily filter unit test projects in the future. Then click 'Next'.

![](2024-04-30-22-08-36.png)

Choose the same Framework as the project being tested (in this case, .NET 6.0). Then click 'Create'.

![](2024-04-30-22-08-56.png)

## 2. Add Project Reference

Add a project reference to the test project for the project to be tested.

![](2024-04-30-22-09-03.png)

![](2024-04-30-22-09-08.png)

![](2024-04-30-22-09-13.png)

## 3. Add NuGet Packages (Optional)

If you want test reports be displayed on GitLab's CI/CD Pipeline web UI, install the `JunitXml.TestLogger` package by following the steps below.

![](2024-04-30-22-09-26.png)

![](2024-04-30-22-09-33.png)

Click 'Browse' -> Enter `JunitXml.TestLogger` in the search box below.

![](2024-04-30-22-09-41.png)

Click 'Install'.

![](2024-04-30-22-09-47.png)

## 4. Update NuGet Packages (Optional But Recommended)

Update the NuGet packages within the project to the latest versions.

![](2024-04-30-22-09-54.png)

![](2024-04-30-22-09-58.png)

![](2024-04-30-22-10-03.png)

![](2024-04-30-22-10-08.png)

## 5. Write Tests

After the setup is complete, you can start writing tests.

Here’s a basic test:

```cs
[Fact]
public void FactUsage()
{

    // Arrange
    var admin = new User();
    var r = new Order();

    // Act
    bool result = r.IsCreatedBy(admin);

    // Assert
    Assert.True(result);
}
```
