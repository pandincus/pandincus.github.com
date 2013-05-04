--- 
layout: post
title: "Configuring Your Dell BIOS Remotely"
author: "dan"
comments: true
tags:
- bios
- remote
- windows
---

This past weekend I wanted to experiment with setting up Team Foundation Build on a VM, and I figured I’d do it at my leisure from the comfort of my home.

After connecting via RDP to my office computer, I realized that I had forgotten to turn on `Hardware-Assisted Virtualization (HAV)` before leaving on Friday. Not a big deal, but annoying to think that my VM [could be working 10-30% faster][1], and that all I’d had to do was change a setting in the BIOS.

I started to wonder if there was some way I could configure my office computer’s BIOS remotely. I looked into a variety of tools, all claiming to give you hundreds of configuration/management options (hardly any of them free, of course). I just wanted a simple tool to [do one thing and do it well][2].

Then I stumbled across the [Dell Client Configuration Utility][3]:

> Dell® Client Configuration Utility lets you create a stand-alone package that you can manually run on a Dell client computer to configure a BIOS, update a BIOS, or capture BIOS settings inventory data.

Granted, it didn’t let me configure the BIOS remotely, but I could certainly use this tool via RDP.

Some stipulations:

* The website lists compatibility as Vista x32 and x64, Server 2003, XP x32 and x64. My office computer is running Windows 7 Pro x64, and I encountered no issues, but YMMV.
* The [DCCU webpage][4] lists the Dell models that are compatible with this tool. **My model was not on this list, but I took a risk.** Again, YMMV.

After installation, it seems that the utility is basically a local ASP.NET website running on port 1000. The interface is completely web-based, and didn’t work properly for me in Google Chrome, so I switched to Internet Explorer:

![Dell Client Configuration Utility Overview][5]

The steps to configure your BIOS are rather simple:

 1. Click the **Create BIOS Inventory Package** button.
 2. A file named **inventory.exe** will be generated. Run this file. *(You may need to run it as an administrator)*
 3. A file named **TaskResult.xml** will be created in the same directory as the **inventory.exe**. This is your computer’s BIOS configuration, in XML format.
 4. Now, back in the web interface, under the **BIOS Settings** heading, browse for the **TaskResult.xml** file and then click the import button to import the settings.

![Importing Collected BIOS Inventory][6]

 5. Now the settings should all load in the grid below. Go ahead and scroll or search for the setting you wanted to change (so in my case, Virtualization).

![Enabling CPU Virtualization in BIOS][7]

 6. Make the change(s), and then click **Create BIOS Settings Package** at the very bottom of the screen.
 7. A file named **settings.exe** will be generated. Run this file. *(You may need to run it as an administrator)*
 8. Restart your computer.
 9. Your BIOS settings should now be configured appropriately!

Running the [HAV Detection Tool][8], I was able to confirm the change:

![Hardware Virtualization Enabled][9]

I just thought this was awesome; I was able to reconfigure my BIOS completely through remote desktop! You can even use this tool to configure a bunch of BIOS settings on several client machines — of course, they should probably all be running the same BIOS version, and you’ll have to run the **settings.exe** file manually on each machine (perhaps through a logon script).

I doubt I’ll ever need to use this tool again, but I think it is handy to know that it exists. I wonder if other computer manufacturers offer similar utilities?


  [1]: http://www.hanselman.com/blog/VirtualPCTipsAndHardwareAssistedVirtualization.aspx
  [2]: http://en.wikipedia.org/wiki/Unix_philosophy
  [3]: http://en.community.dell.com/techcenter/systems-management/w/wiki/1975.dell-client-configuration-utility.aspx
  [4]: http://support.us.dell.com/support/downloads/download.aspx?c=us&l=en&s=gen&releaseid=R204280&formatcnt=1&libid=0&fileid=285029
  [5]: /img/blog/dccu_1.png
  [6]: /img/blog/dccu_2.png
  [7]: /img/blog/dccu_3.png
  [8]: http://www.microsoft.com/downloads/en/details.aspx?FamilyID=0ee2a17f-8538-4619-8d1c-05d27e11adb2
  [9]: /img/blog/dccu_4.png