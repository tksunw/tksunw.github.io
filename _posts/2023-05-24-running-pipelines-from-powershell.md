---
layout: post
title: "Using PowerShell to run Azure DevOps Pipelines"
categories: azure, powershell
---

We have some Azure DevOps pipelines that run some comparisons between Azure SQL Databases running in production
and their DR replicas.  The pipelines do things like make sure the number of objects in each database are the same, 
or within a predefined tolerance (as replication can (and will) lag).  No problem if you're having to manually 
compare 1 database between two sites.  But when you're running a full set of comparisons over several servers 
and hundreds of databases between several sites/tiers/sandboxes, it can be time consuming.  

So time consuming, in fact, that my boss decided he didn't want to do it anymore, and kindly dropped it on me. :)

Usually, the pipeline is run manually and requires a number of Parameters to be set, such as 'Test Name', 
'Test Description', 'Application' (that's the db being compared), 'Primary Site', 'Secondary Site'.  If we want to 
run a full suite against even just one of our Azure SQL Servers, this could take quite a while as you have to load 
the page in the browser, click 'Run Pipeline', fill out text fields, make appropriate selections, and click 'Run'.  

![Pipeline Parameters in Azure DevOps](/images/run-pipeline-parameters.png)

So.  If you can run it from a browser, you can run it from the CLI.  In the old days, that might have involved 
using HTTP-LiveHeaders to deconstruct HTTP calls and reverse engineering some functionality.  Now, we just need 
PowerShell and `Invoke-WebRequest`, since Azure DevOps has a pretty robust REST API.

---

I found most of the parts of this in various answers on StackOverflow sites, Microsoft Communities questions, and Reddit.
But I didn't see anywhere that put them all together in one place.

#### The first thing you need is an Azure DevOps Personal Access Token

The PAT you create needs to have enough permissions to manage build and release pipelines, and you can follow
the steps at [Microsoft Learn: Use personal access tokens](https://learn.microsoft.com/en-us/azure/devops/organizations/accounts/use-personal-access-tokens-to-authenticate?view=azure-devops&tabs=Windows).

Once you have the PAT, to use it in an Authentication header with calls to the REST API, it needs to be encoded as
a BASE64 string.  To make things easier on myself I usually keep things tokens like the Az DevOps PAT as environment
variables. Then I encode it, and build a hashtable of headers I can use with any `Invoke-WebRequest` calls. 

```powershell
# my PAT is in the environment variable `buildPAT`
$env:buildPAT = 'xm3ij2tazssbacc3442pgazxalb6kge3mze2rt3zcmob5w7wjwyq'

$authHeader = 'BASIC ' + [Convert]::ToBase64String([Encoding]::ASCII.GetBytes(":$($env:buildPAT)")

$iwrParams = @{
    Headers = @{
        'Content-Type'  = 'application/json'
        'Authorization' = $authHeader
    }
}
```
Now `Invoke-WebRequest @iwrParams -Uri <any azdo uri> -Method Get` should work. Feel free to stick your Method 
and Uri into the `$iwrParams` hashtable, too.  You can always change them later if you need to `POST`, or use 
a different Uri.

#### Using your PAT with your Azure DevOps Project

Now, armed with your trusty PAT, you can set up some basic variables for your Organization and Project, that
we will use to build the URIs we'll use with our requests.

```powershell
$myOrg      = 'StainlessSteelNetworks'
$myProject  = 'DRValidation'
$myPipeLine = 'DBCompare'

$plBaseUrl  = "https://dev.azure.com/{0}/{1}/_apis/pipelines" -f $myOrg, $myProject
$plApiUrl   = "${plBaseUrl}?api-version=7.0"
```

From there, we can do things like list out all the pipeline in the project, filter by name, and match just the
one we want to run.  And from that, we can get the pipeline ID, which we'll need to trigger the run.

```powershell
$allPipelines = Invoke-WebRequest @iwrParams
$runPipeline  = ($allPipeline.Content | ConvertFrom-Json).Value | Where-Object {$_.Name -eq $myPipeline}

$runPlId      = $runPipeline.id
$runPlVer     = $runPipeline.revision
$runPlUrl     = "${plBaseUrl}/${runPlId}/runs?pipelineVersion=${runPlVer}&api-version=7.0"
```

#### Remember those parameters?

It turns out passing the parameters to the `$runPlUrl` is pretty easy to do.  You just need to build a proper 
JSON body for your POST that includes your Parameters in a `templateParameters` stanza.  [Microsoft Learn: Runs - Run Pipeline](https://learn.microsoft.com/en-us/rest/api/azure/devops/pipelines/runs/run-pipeline?view=azure-devops-rest-7.0)

For me, since I'm running the pipeline in the `main` branch, I specify that in the first block.  Then the
second block contains the `templateParameters` that we'd normally have to fill out in the browser.

```powershell
$auditName        = 'InternalTestQ2'
$auditDescription = 'Internal tests performed in Q2 by TK'
$auditWhichApp    = 'lunch-order'
$auditPriEnv      = 'PRD'
$auditSecEnv      = 'DR'
$json = @"
    {
      "resources": {
        "repositories": {
          "self": {
            "refName": "main"
          }
        }
      },
      "templateParameters": {
        "auditName": "$auditName",
        "auditDescription": "$auditDescription",
        "auditWhichApp": "$auditWhichApp",
        "environmentPrimary": "$auditPriEnv",
        "environmentSecondary": "$auditSecEnv",
      }
    }
"@
```

I think that's the bit that isn't too clear in the documentation, or the various examples out there on the internet.

#### Running the Pipeline

Once you have all those pieces set, running the actual pipeline is quite trivial.

```powershell
$plRun = Invoke-WebRequest @iwrParams -Method Post -Uri $runPlUrl
```

That's it.  Your pipeline is running, and you can watch it in the Azure DevOps web portal.

Now, if you want, you can take it one step further and monitor your pipeline for completion and success/failure.

```powershell
$plStatusUrl = ($plRun.Content | ConvertFrom-Jason).Url
$jobStatus   = Invoke-WebRequest @iwrParams -Method Get -Uri $plStatusUrl
while (($jobStatus.Content | ConvertFrom-Json).State -ne 'completed') {
    start-sleep -seconds 10
    $jobStatus = Invoke-WebRequest @iwrParams -Method Get -Uri $plStatusUrl
}

$result          = $jobStatus.Content | ConvertFrom-Json
$elapsedTimeSpan = New-Timespan -Start $result.createdDate -End $result.finishedDate
$elapsedTime     = "{0:dd}d:{0:hh}h:{0:mm}m:{0:ss}s" -f $elapsedTimeSpan
$finalResult     = $result.result

Write-Host ('-' * 80)
Write-Host @"
Pipeline:     $myPipeline
Description:  ${auditName}: ${auditDescription}
Application:  $auditWhichApp
Result:       $finalResult
Elapsed Time: $elapsedTime
"@
Write-Host ('-' * 80)
```

When it's done, you get a nice little output like this:
```powershell
--------------------------------------------------------------------------------
Pipeline:     DBCompare
Description:  InternalTestQ2: Internal tests performed in Q2 by TK
Application:  lunch-order
Result:       succeeded
Elapsed Time: 00d:00h:02m:43s
--------------------------------------------------------------------------------
```

I won't bore you with turning this into an advanced function, or writing a loop 
to loop through all the valid applications, or doing parameter validation.  
