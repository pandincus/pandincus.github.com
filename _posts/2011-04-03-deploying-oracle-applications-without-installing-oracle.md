--- 
layout: post
title: "Deploying Oracle Applications without Installing Oracle"
author: "dan"
comments: true
tags:
- .net
- deploying
- oracle
---

Depending on your environment, getting an app running on a production server can sometimes be a daunting task. Generally, the less you have to configure, the better. So when you need to deploy an application that uses [Oracle.DataAccess][1] and the right version of Oracle isn’t installed, good luck!

We shouldn’t blame the server admins. Maybe they are overloaded with tasks and simply don’t have the time, or maybe they’re worried that installing a new version of Oracle will break other functionality. Whatever the case, we have to understand their position.

Anyway, I had a similar situation with a recent .NET project I was working on. Fortunately, after doing a bit of research (and being very stubborn), I found a fantastic approach that requires no installation of Oracle on the production machine whatsoever. You simply copy a few *.dlls into your project, push the build, and watch the application run.

The below steps are using ODAC 11.2 Release 3 (11.2.0.2.1):

 1. Install the latest version of ODAC on your development machine.
 2. From your oracle client home, grab `oci.dll` and `oraociei11.dll`.
 3. From the **\bin** folder inside your oracle client home, grab `OraOps11w.dll`.
 4. Add the above three .dlls to your Visual Studio project.
 5. From the properties window, make sure that the **Build Action** is set to **Content** and the **Copy to Output Directory** is set to **Copy if newer**.
 6. Make sure your project is referencing `Oracle.DataAccess`, which it should be anyway if you’re using ODAC in your project.
 7. The **Copy Local** property for `Oracle.DataAccess` should be set to **True**.

That’s it! Push your project, and you should see a total of **four** Oracle-related .dll files get copied into your output folder. Give this to your server admins and tell them you knocked one item off their todo list ;-)

*Note:* `oraociei11.dll` is approximately 120MB, which stinks, but isn’t a bad sacrifice to make.

If your team needs to do this on a large number of projects, you could do something neat like [create a local NuGet package][2]. I have a blog post about this coming soon.

Credits:

 * [What is the minimum client footprint required to connect C# to an Oracle database? – Stack Overflow][3]
 * [Using the new ODP.Net to access Oracle from C# with simple deployment - Chris Hulbert][4]


  [1]: http://www.oracle.com/technetwork/database/windows/downloads/index-101290.html
  [2]: http://haacked.com/archive/2010/10/21/hosting-your-own-local-and-remote-nupack-feeds.aspx
  [3]: http://stackoverflow.com/questions/70602/what-is-the-minimum-client-footprint-required-to-connect-c-sharp-to-an-oracle-da
  [4]: http://www.splinter.com.au/using-the-new-odpnet-to-access-oracle-from-c/