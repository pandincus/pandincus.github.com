--- 
layout: post
title: "Combining NuGet with Team Foundation Build"
author: "dan"
comments: true
tags:
- .net
- deploying
- nuget
- tfs
- continuous integration
- source control
---

**Update: This post refers to NuGet 1.2. Although the techniques described here are still valid, once NuGet 1.4 comes out, Dan Turner’s fork will no longer be necessary. Thus, please keep the date of this post in mind!**

I’ve been experimenting with setting up a build server for our dev team at work. I didn’t run into any challenges until I configured the build agent.

When I initiated a test build for one of our apps, it failed with a bunch of these:

> Could not resolve this reference. Could not locate the assembly “<assembly>”. Check to make sure the assembly exists on disk. If this reference is required by your code, you may get compilation errors.

D’oh! Of course, **my build agent had no idea where to find my project’s external references**.

This is a common scenario, and there are all kinds of suggested solutions. For referencing your own projects, the Microsoft Patterns & Practices group [suggests the following][1]:

> …you need to get the source or assemblies from that project into the workspace on your build server…edit your TFSBuild.proj file to add the assembly or the solution reference and override the BeforeGet event to get assemblies or sources from each of the team projects on which you have a dependency.

The above solution, however, would always **get the latest version** of your referenced projects. What if you wanted a specific version? Microsoft Patterns & Practices suggests [branching the external team project into your team project][2].

Although branching and utilizing source control makes sense, putting third-party, compiled assemblies into source control puts a bad taste in my mouth. Surely there must be some other way? A search on StackOverflow reveals a lot of people [asking][3] about [how][4] to do [this][5], and a lot of different approaches to solving it.

## Enter NuGet

[NuGet][6] is a relatively new, open-source package management solution for .NET developers that has been getting a lot of support from Microsoft. (The idea is to make the process of adding external, third-party libraries easier — here’s [an example][7].)

Our team has been slowly adopting the use of NuGet for all of our third party libraries. Now that we have a build server up and running, I was wondering: **Can we leverage NuGet to resolve our dependencies for us?**

Daniel Auger *briefly* discusses using NuGet for .NET Dependency Management, but [he dismisses it entirely][8] in his blog:

> **Nuget is not the final answer for teams using TFS**

> I’ve been following the Nuget dev list closely, and Nuget is considered to be a development time dependency resolver only, not a build time resolver. This means that if you use TFS to track your Nuget package folders, you could still run into dll versioning issues.

<s>I admit I don’t understand Daniel’s point here. NuGet should be able to track versions for us, and the versioning should work just as well on our build server as on our development machines.</s>

**Edit:** I asked Daniel about this and, after further discussion, he believes NuGet can be used as a build time resolver as well.

To understand how we can use NuGet, let’s look at what happens when we add a package to a typical .NET project:

    MySolution\
      MyProject\
        bin\
        obj\
        Properties\
        MyProject.csproj
        Program.cs
      MySolution.sln

Now let’s add, for example, Ninject v 2.2.1.0:

    MySolution\
      MyProject\
        bin\
        obj\
        Properties\
        MyProject.csproj
        Program.cs
        packages.config
      packages\
        Ninject.2.2.1.0\
          lib\
            (the assemblies go here)
          Ninject.2.2.1.0.nupkg
        repositories.config
      MySolution.sln

* The **packages.config** file exists once per project, and it lists any NuGet packages that are in use for that project (it also includes the version #):

<pre class="brush: xml;">
<packages>
  <package id="Ninject" version="2.2.1.0" />
</packages>
</pre>

* The **repositories.config** file just points to each **packages.config** file for the solution.
* The **packages\** folder is where all downloaded assemblies for any projects in the solution are stored.
* The **MyProject.csproj** file is changed to add the actual assembly to your project:

<pre class="brush: xml;">
<Reference Include="Ninject">
    <HintPath>..\packages\Ninject.2.2.1.0\lib\.NetFramework 4.0\Ninject.dll</HintPath>
</Reference>
</pre>

We can leverage the [command line NuGet.exe program][9] to download NuGet packages from the online repository to the local **packages\\** folder, just by pointing it at our **packages.config** file! Here’s the plan:

 1. Each developer would install **NuGet.exe** on their system path.
 2. We’d download **NuGet.exe** to the build server and add it to the system `PATH`.
 3. When developing any project that used NuGet references, we’d add

<pre class="brush: ps">
nuget install “$(ProjectDir)packages.config” -o “$(SolutionDir)packages”
</pre>

to the project’s Pre-build event. *(This will tell NuGet to install any packages listed in the project’s packages.config, and download their files to the ‘packages’ folder under the solution folder.)*

Almost perfect. But what if we needed references that weren’t on NuGet? Perhaps we had to reference some obscure vendor .dll that had no place in a public gallery. To handle this case, we had to create a local NuGet repository and agree to store all internal .dlls there.

But wait! The `nuget install` command only lets us specify **one** source for our libraries; meaning we can either specify the public NuGet repository or our local repository, but not both.

Fortunately, in [a discussion on CodePlex][10], [Dan Turner][11] decided to be amazing and [create a fork of NuGet.exe that allows you to specify multiple sources][12]. This makes our entire Pre-build event:

<pre class="brush: ps">
nuget install “$(ProjectDir)packages.config” -o “$(SolutionDir)packages”
-s “\\asa01\asa_share\Dan\ASA NuGet Repository”;https://go.microsoft.com/fwlink/?LinkID=206669
</pre>

This solution, to me, feels perfect. We’re not storing any assemblies in source control, and our **packages.config** file should handle package versioning for us.

A big thank you to Dan Turner for making those changes to NuGet.exe. *(Perhaps it’ll get added to trunk?)* Hopefully this article will guide someone else in a similar situation!

  [1]: http://tfsguide.codeplex.com/wikipage?title=Practices%20at%20a%20Glance:%20%20%20Build
  [2]: http://tfsguide.codeplex.com/wikipage?title=Chapter%206%20-%20Managing%20Source%20Control%20Dependencies%20in%20Visual%20Studio%20Team%20System&ProjectName=tfsguide
  [3]: http://stackoverflow.com/questions/3242663/team-foundation-server-2010-build-with-external-library
  [4]: http://stackoverflow.com/questions/1823594/team-foundation-build-resolving-cross-solution-project-references
  [5]: http://stackoverflow.com/questions/1207728/team-foundation-server-add-rereference-to-existing-dll-to-a-new-class-library-p
  [6]: http://nuget.codeplex.com/
  [7]: http://haacked.com/archive/2010/10/06/introducing-nupack-package-manager.aspx
  [8]: http://mynerditorium.blogspot.com/2011/02/net-dependency-management-in-pre-nuget.html
  [9]: http://nuget.codeplex.com/releases/58939/download/222685
  [10]: http://nuget.codeplex.com/discussions/249628
  [11]: http://www.codeplex.com/site/users/view/dant199
  [12]: http://nuget.codeplex.com/SourceControl/network/Forks/dant199/NuGetMultipleSources