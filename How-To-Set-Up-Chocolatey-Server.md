# How To Setup the Chocolatey.Server Package

<!-- TOC -->

- [What is Chocolatey.Server?](#what-is-chocolateyserver)
- [Requirements](#requirements)
- [Setup with Puppet](#setup-with-puppet)
- [Setup Normally](#setup-normally)
- [Additional Configuration](#additional-configuration)
  - [Performance](#performance)
    - [Future](#future)
- [Common Errors and Resolutions](#common-errors-and-resolutions)
  - [Error 404 on Push](#error-404-on-push)
  - [Error 500](#error-500)
  - [Error 500.19 - "The requested page cannot be accessed because the related configuration data for the page is invalid."](#error-50019---the-requested-page-cannot-be-accessed-because-the-related-configuration-data-for-the-page-is-invalid)
  - [Other error](#other-error)

<!-- /TOC -->

**NOTE:** Refer to [[How To Set Up Chocolatey For Organizational/Internal Use|How-To-Setup-Offline-Installation]] in tandem with this article.

## What is Chocolatey.Server?
The [Chocolatey.Server package](https://chocolatey.org/packages/chocolatey.server) contains the binaries for a fully ready to go Chocolatey NuGet Server where you can serve packages over HTTP using a NuGet-compatible OData feed.

[Chocolatey Server](https://chocolatey.org/packages/chocolatey.server) is a simple Nuget.Server that is ready to rock and roll. It has already completed Steps 1-3 of NuGet's [host your own remote feed](https://docs.nuget.org/Create/Hosting-Your-Own-NuGet-Feeds#creating-remote-feeds). Version 0.1.2 has the following additional adds:

* Uses same enhanced NuGet that Chocolatey uses so you can see more information in search if you choose to use those things.
* Allows packages up to 2GB. Package size can be controlled through [maxAllowedContentLength](https://msdn.microsoft.com/en-us/library/ms689462(v=vs.90).aspx) and [maxRequestLength](https://msdn.microsoft.com/en-us/library/e1f13641(v=vs.100).aspx).

When you install it, it will install the website typically to `c:\tools\chocolatey.server`.

## Requirements

* You need a Windows box with at least 50GB of free space (or where ever you are going to put the packages).
* 50GB of free space for where ever you will put packages.
* We recommend at least 8GB RAM, but more if you can.
* Ability to set up an IIS site and unblock website ports.
* If you have an IIS site for WSUS administration, Chocolatey.Server website will not come up at all, even if everything looks right. We have not yet been able to determine the issue, but believe it is related to ASP.NET 4.6+. Installing all of the required components for Chocolatey.Server may also affect your WSUS admin site. Please seek a different box.
* If you can ensure your server is up to date with all of the Windows Updates, you will move through this process quite a bit quicker.

## Setup with Puppet
If you are using the Puppet module [chocolatey/chocolatey_server](https://forge.puppet.com/chocolatey/chocolatey_server), it will do all of the additional setup for this package and allow some customization.

The module works with Windows Server 2008/2012.
For a simple `include chocolatey_server` it does the following automatically:

 * Ensures IIS is installed
 * Ensures ASP.NET is installed
 * Ensures the chocolatey.server package is installed
 * Ensures an app pool for the chocolatey.server site
 * Ensures the IIS website is setup for chocolatey.server
 * Ensures permissions for the site are set correctly.

## Setup Normally
 * If your Windows updates are not up to date, there are two required Windows updates you are going to need (heads up they take awhile)
    * Install KB2919355 - `choco install KB2919355 -y` - this one or the other Windows update takes a ***very*** long time to install, just be patient
    * Restart your machine.
    * Install KB2919442 - `choco install KB2919442 -y` (IIRC this is the one that takes forever...) -
    * Reboot that machine again
 * You need at least .NET Framework 4.6. If you don't have that or newer, then run `choco install dotnet4.6.1 -y`
    * Reboot one more time, thanks Windows!!
 * Install or upgrade the package - `choco upgrade chocolatey.server -y`
 * Ensure IIS is installed. You can try `choco install IIS-WebServer --source windowsfeatures`
 * Ensure that ASP.NET is installed. Try `choco install IIS-ASPNET45 --source windowsfeatures` (Windows Server 2012). Use `IIS-ASPNET` for Windows Server 2008, possibly `IIS-ASPNET46` for Windows Server 2016.
 * Disable or remove the Default website
 * Set up an app pool for Chocolatey.Server. Ensure 32-bit is enabled and the managed runtime version is `v4.0` (or some version of 4). Ensure it is "Integrated" and not "Classic".
 * Set up an IIS website pointed to the install location and set it to use the app pool.
 * Go to explorer and right click on `c:\tools\chocolatey.server` and add the following permissions:
   * `IIS_IUSRS` - Read
   * `IUSR` - Read
   * `IIS APPPOOL\<app pool name>` - Read
 * Right click on the `App_Data` subfolder and add the following permissions:
   * `IIS_IUSRS` - Modify
   * `IIS APPPOOL\<app pool name>` - Modify

## Additional Configuration

Looking for where the apikey is and how it is changed, that is all done through the web.config. The config is pretty well-documented on each of the appSettings key values.

To update the apiKey, it is kept in the web.config, search for that value and update it. If you reach out to the server on https://localhost (it will show you what the apikey is, only locally though).

### Performance

To configure for performance, you will want to do the following:

* Keep the site warm (https://serverfault.com/a/595215/79259):
  * Turn on Application Initialization in Windows Features under Web Server (IIS) -> Web Server -> Application Development -> Application Initialization (you can also try `choco install IIS-ApplicationInit --source windowsfeatures`)
  * Application Pool Advanced Settings:
    * General: Start Mode -> Always Running
    * Process Model: Idle Time-out (minutes) -> 0
    * Recycling: Regular Time Interval (minutes) -> 0
  * Under the Site's Advanced Settings:
    * Preload Enabled -> True


#### Future

We are looking to add support for the package source to automatically handle this aspect - http://blog.nuget.org/20150922/Accelerate-Package-Source.html


## Common Errors and Resolutions

When you are attempting to install the Simple Server, you may run into some errors depending on your configuration. Here are some common ones we've seen that you may get when you browse to the the site.

### Error 404 on Push
If you can do everything except push packages, it is likely your application pool is set to Classic mode and can't find directories. It needs to be "Integrated". Please change that to Integrated and then recycle the application pool. That should resolve the issue of pushing packages. Reference: https://stackoverflow.com/a/37702935/18475.

### Error 500

Take a closer look at the error. It may be one of the other errors below.

### Error 500.19 - "The requested page cannot be accessed because the related configuration data for the page is invalid."

This can mean a couple of things:

* You missed ensuring the website is using an app pool that is at least .NET 4.0. Check the app pool that your site is using, then ensure that app pool has `32-bit` enabled and the managed runtime version is `v4.0` (or some version of 4).
* You have made a change to the xml file and it is not valid xml. This typically happens if you put an xml escape character into the password (`&`). To do that you would need to set CData around the value or use a different password. It could also happen if you accidentallly change the xml and it is no longer valid.
* You are attempting to set up Chocolatey Server next to WSUS Administration website. For an unknown reason, something won't register correctly with Chocolatey Server and its need for ASP.NET 4.6+. So we recommend not putting the Chocolatey Server next to that website. Find a machine with the WSUS administration site.

### Other error

Turn on customErrors under system.web - <customErrors mode="Off" /> - see this guide to set it - https://stackify.com/web-config-customerrors-asp-net/

Then browse to the site to see if you can gather any more information.

If so, and you are a commercial edition customer, please open a support ticket. If you are using open source Chocolatey, please open a ticket at https://github.com/chocolatey/simple-server/issues.

