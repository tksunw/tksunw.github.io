---
title: "PowerShell on Linux"
date: 2017-07-24 12:00:00 -0500
categories: [Linux]
tags: [powershell, linux, ubuntu]
---

I've been working with Windows and VMware for a while now, and have really enjoyed learning PowerShell and PowerCLI. I've always preferred CLI tools to GUI tools. Possibly just because I'm old enough that the computers I started with didn't have Windows (or even X-Windows).

The more I use PowerShell, the more I like PowerShell, so I've decided to start managing the Linux servers I have at home with it, just for funsies.

The first step is to install PowerShell. PowerShell for Linux/Mac/Etc is v6, and still in beta at the time of this writing. I use Ubuntu Linux at home, and fortunately for my lazy self, there is an Apt Repo for PowerShell for Ubuntu 16.

```bash
curl https://packages.microsoft.com/keys/microsoft.asc | sudo apt-key add -
curl https://packages.microsoft.com/config/ubuntu/16.04/prod.list | sudo tee /etc/apt/sources.list.d/microsoft.list
sudo apt-get update
sudo apt-get install -y powershell
```

These steps were taken from the actual Ubuntu 16.04 Installation Instructions. If you aren't comfortable adding the repository, there are also instructions for manually downloading the .deb package and installing it.

```powershell
tkennedy@vp-win10tk01:~$ powershell
PowerShell v6.0.0-beta.4
Copyright (C) Microsoft Corporation. All rights reserved.

PS /home/tkennedy> $PSVersionTable

Name                           Value
----                           -----
PSVersion                      6.0.0-beta
PSEdition                      Core
GitCommitId                    v6.0.0-beta.4
OS                             Linux 4.4.0-43-Microsoft #1-Microsoft Wed Dec 31 14:42:53 PST 2014
Platform                       Unix
PSCompatibleVersions           {1.0, 2.0, 3.0, 4.0...}
PSRemotingProtocolVersion      2.3
SerializationVersion           1.1.0.1
WSManStackVersion              3.0
```

In UNIX and Linux, everything is a file. The shell (bash, zsh, etc) kind of turn all those files into strings, and make them available to STDOUT, which can be used on a pipeline to do things like `command | sed | awk | wc` or something. In PowerShell, everything is an object, which can be very powerful, but which can also be overwhelming as you're trying to get used to having to understand each object's model. They are rarely the same.

## How many files are there in this directory?

**Linux:**

```bash
PS /home/tkennedy> find . -type f | wc -l
13
```

**PowerShell:**

```powershell
PS /home/tkennedy> (Get-ChildItem -Force -Recurse -File).Count
13
```

The really interesting piece here, is that if I want to do something with those files, on the unix side I have to parse the list of files that `find` gives me back, and then process each file to, say, get its `stat` results, or something. Then I have to further process all that data. Because everything is a string.

With PowerShell, I can assign the results of the `Get-ChildItem` command to a variable, `$files`, and it will create a `System.Array` containing all the objects for the files that were found.

```powershell
PS /home/tkennedy> $files = Get-ChildItem -Force -Recurse -File
```

Now, because everything is an object, the `$files` object that I created is basically an array of all the file objects that `Get-ChildItem` was able to identify, and each of those file objects has all the properties that corresponds to that type of object.

If I wanted a list of files, their sizes, and their modes:

```powershell
PS /home/tkennedy> $files | Select Name, Mode, Length

Name                           Mode   Length
----                           ----   ------
.bash_history                  ---h--    263
.bash_logout                   ---h--    220
.bashrc                        ---h--   3771
.profile                       ---h--    655
.sudo_as_admin_successful      ---h--      0
.viminfo                       ---h--   3601
appstacks.json                 ------  45413
appstacks.py                   ------    247
ModuleAnalysisCache            --r---  36043
StartupProfileData-Interactive --r---  17788
nuget.config                   --r---     93
PSRepositories.xml             --r---   3498
ConsoleHost_history.txt        ------   1128
```

To see what kinds of properties are available in a PowerShell object, you can use the `Get-Member` cmdlet on the pipeline. This will show you all the member properties of the `System.IO.FileInfo` objects.

```powershell
PS /home/tkennedy> $files | Get-Member

   TypeName: System.IO.FileInfo

Name                      MemberType     Definition
----                      ----------     ----------
LinkType                  CodeProperty   System.String LinkType{get=GetLinkType;}
Mode                      CodeProperty   System.String Mode{get=Mode;}
Target                    CodeProperty   System.Collections.Generic.IEnumerable...
AppendText                Method         System.IO.StreamWriter AppendText()
CopyTo                    Method         System.IO.FileInfo CopyTo(string destFileName)...
Create                    Method         System.IO.FileStream Create()
CreateText                Method         System.IO.StreamWriter CreateText()
Decrypt                   Method         void Decrypt()
Delete                    Method         void Delete()
Encrypt                   Method         void Encrypt()
Equals                    Method         bool Equals(System.Object obj)
GetHashCode               Method         int GetHashCode()
GetType                   Method         type GetType()
MoveTo                    Method         void MoveTo(string destFileName)
Open                      Method         System.IO.FileStream Open(System.IO.FileMode mode)...
OpenRead                  Method         System.IO.FileStream OpenRead()
OpenText                  Method         System.IO.StreamReader OpenText()
OpenWrite                 Method         System.IO.FileStream OpenWrite()
Refresh                   Method         void Refresh()
Replace                   Method         System.IO.FileInfo Replace(string destinationFileName...)
ToString                  Method         string ToString()
PSChildName               NoteProperty   string PSChildName=.bash_history
PSDrive                   NoteProperty   PSDriveInfo PSDrive=/
PSIsContainer             NoteProperty   bool PSIsContainer=False
PSParentPath              NoteProperty   string PSParentPath=...
PSPath                    NoteProperty   string PSPath=...
PSProvider                NoteProperty   ProviderInfo PSProvider=...
Attributes                Property       System.IO.FileAttributes Attributes {get;set;}
CreationTime              Property       datetime CreationTime {get;set;}
CreationTimeUtc           Property       datetime CreationTimeUtc {get;set;}
Directory                 Property       System.IO.DirectoryInfo Directory {get;}
DirectoryName             Property       string DirectoryName {get;}
Exists                    Property       bool Exists {get;}
Extension                 Property       string Extension {get;}
FullName                  Property       string FullName {get;}
IsReadOnly                Property       bool IsReadOnly {get;set;}
LastAccessTime            Property       datetime LastAccessTime {get;set;}
LastAccessTimeUtc         Property       datetime LastAccessTimeUtc {get;set;}
LastWriteTime             Property       datetime LastWriteTime {get;set;}
LastWriteTimeUtc          Property       datetime LastWriteTimeUtc {get;set;}
Length                    Property       long Length {get;}
Name                      Property       string Name {get;}
BaseName                  ScriptProperty System.Object BaseName {get=...}
VersionInfo               ScriptProperty System.Object VersionInfo {get=...}
```

All of those properties are available to you, and to other cmdlets in PowerShell to parse, filter on, operate on, print out, etc, etc. The shell is your oyster!
