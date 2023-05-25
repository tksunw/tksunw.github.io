---
layout: post
title: "Using PowerShell to run Azure DevOps Pipelines"
categories: azure, powershell
---

We have some Azure DevOps pipelines that run some comparisons between Azure SQL Databases running in production
and their DR replicas.  The pipelines do things like make sure the number of objects in each database are the same,
or within a predefined tolerance (as replication can (and will) lag).  No problem if you're comparing 1 database between two sites.  But when you're running a full set of comparisons over several servers and hundreds of databases between several sites/tiers/sandboxes, it can be time consuming.  So time consuming, in fact, that my boss decided he didn't want to do it anymore, and kindly dropped it on me. :)

Usually, the pipeline is run manually, and requires a number of Parameters to be set, such as 'Test Name', 'Test Description', 'Application' (that's the db being compared), 'Primary Site', 'Secondary Site'.  If we want to run a full suite against even just one of our Azure SQL Servers, this could take quite a while.  Load the page in the browser, click 'Run Pipeline', fill out text fields, make appropriate selections, and click 'Run'.  

![Pipeline Parameters in Azure DevOps](/images/run-pipeline-parameters.png)

So.  If you can run it from a browser, you can run it from the CLI.  In the old days, that might have involved using HTTP-LiveHeaders to deconstruct HTTP calls and reverse engineering some functionality.  Now, we just need PowerShell and `Invoke-WebRequest`, since Azure DevOps has a pretty robust REST API.



---

### This is a header

#### Some T-SQL Code

```tsql
SELECT This, [Is], A, Code, Block -- Using SSMS style syntax highlighting
    , REVERSE('abc')
FROM dbo.SomeTable s
    CROSS JOIN dbo.OtherTable o;
```

#### Some PowerShell Code

```powershell
Write-Host "This is a powershell Code block";

# There are many other languages you can use, but the style has to be loaded first

ForEach ($thing in $things) {
    Write-Output "It highlights it using the GitHub style"
}
```
