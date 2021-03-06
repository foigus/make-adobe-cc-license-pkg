# make-adobe-cc-license-pkg

## Overview

This is a command-line tool that makes it easier to deploy Adobe Creative Cloud device license files (output by the Creative Cloud Packager application) on OS X, by building them into a standard OS X package installer.

Given a path to the license file output from Creative Cloud Packager, this tool packages the `adobe_prtk` executable and adds a postinstall script to perform the activation. It can also import the package into Munki with an appropriate uninstaller script added so that the license can be deactivated simply by removing the item using Munki. If not using Munki, the uninstall script is output alongside the package for your own use.

## Why?

Here's a [blog post](http://macops.ca/adobe-creative-cloud-deployment-packaging-a-license-file).

Adobe Creative Cloud Packager offers an option "Create License File," but does not make it obvious about how to deploy the license activation, and there's no tool for deactivating the license with a CC For Teams license. The `RemoveVolumeSerial` tool output by CCP can only be used for those with Enterprise agreements. (An enterprise LEID is hardcoded in the binary.) There is some very confusing documentation available [here](https://helpx.adobe.com/creative-cloud/packager/create-license-file.html), [here](https://helpx.adobe.com/creative-cloud/packager/provisioning-toolkit-enterprise.html), and more details about LEIDs [here](https://helpx.adobe.com/content/help/en/creative-cloud/packager/creative-cloud-licensing-identifiers.html).

Why would you even want to make a license file, when there's already an option to build device licensed packages? After all, these packages will handle activating a device license, and Munki has support for running CCP's uninstaller packages to remove the application. Unfortunately, uninstalling a device-licensed package built with CCP removes that device license activation _regardless_ of whether other applications also relying on that activation are installed, making it a pain to try and manage multiple apps within a single device pool independently of one another.

This behaviour where uninstalling a licensed application package also uninstalls the license (even if other applications may require it) was classified by Adobe as a bug and scheduled to be fixed. The bug was first [documented in April 2015](https://twitter.com/Adobe_ITToolkit/status/591361032905433088), and it has not yet been fixed.

So instead, it seems possible to install a Named License and "convert" it to a device license using a license file and Adobe's activation tools. This might be a more manageable deployment scenario for your environment, regardless of the bug described above, because it allows you to build a single Named installer and then optionally apply a device license to that machine (and remove it later to restore it to a Named license).

With Munki, you could then assign this license pkg to a manifest directly, or potentially add it as an `update_for` multiple CC products with Named licenses. This way, Munki will keep the license around for as long as at least one CC app (for which this is an `update_for` is installed on the system). Munki will deactivate the license if the last dependent CC app is removed (using Munki).

## Requirements

This tool only requires the output from the Creative Cloud Packager's "Create License File" workflow.

## Usage

Run the command with a single argument: the path to a directory containing the output of a "Create License File" workflow from CCP:

```
./make-adobe-cc-license-pkg path/to/ccp/license/files/dir
```

Run the command with the `-h` (or `--help`) option to print out the full usage. There are multiple customization options for package parameters.

If you've also used the `--munki` option to import the package into Munki, you will likely want to modify some other pkginfo keys in the resultant pkginfo file, such as `display_name`, `description`, etc. This script isn't meant to be an all-encompassing Munki importer tool that passes any other options through to `munkiimport`, just enough to import a functional item into Munki with the appropriate `uninstall_*` keys.

## What's it actually do?

1. `helper.bin` (which is really the `adobe_prtk` tool) is scanned to extract a version number so it can be installed in a path that does not conflict with other versions in the future.
1. `prov.xml` is read to extract the [LEID](https://helpx.adobe.com/content/help/en/creative-cloud/packager/creative-cloud-licensing-identifiers.html), which is needed if performing an uninstallation (deactivation) later.
1. Stages your copies of adobe_prtk and prov.xml to be installed to `/usr/local/bin/adobe_prtk_$version/adobe_prtk` and `/private/tmp/prov.xml`, respectively.
1. A Python-based postinstall script is written for the package, which executes `adobe_prtk` with the required options, and removes the prov.xml file. Additionally, if the command emits an error code, the error code along with an explanation (taken from Adobe's documentation) is printed to the install log (!).
1. Similarly, the uninstaller script is written to disk, which executes the appropriate `adobe_prtk` options, using the LEID that was extracted from the `prov.xml` earlier. It also forgets the package receipt and removes the copy of `adobe_prtk` that was installed.
1. If the `--munki` option is given, the pkg will be imported into Munki, with the uninstall script set as the `uninstall_script`.
1. The resultant pkg and uninstall script will be written to the directory specified by `--output` or if omitted, the current working directory.

## More documentation

I've written a [series of blog posts](https://macops.ca/tag/creative-cloud) on deploying Creative Cloud using Munki, which includes considerations on deploying licenses.

At [MacSysAdmin 2015](http://macsysadmin.se/2015/Home.html), I gave a [recorded video session](http://docs.macsysadmin.se/2015/video/Day1Session4.mp4) on Mac Admin tools that included an explanation of deploying Adobe CC using Munki, aamporter and make-adobe-cc-license-pkg (see 12:00-37:30 in the video).

## Thanks

Thanks to [James Stewart](https://github.com/jgstew) for pointing out that the `helper.bin` file is identical to `adobe_prtk`. Previously this tool used to look for the undocumented location where `adobe_prtk` is installed along with CCP, or require you to include your own copy of the tool. The script is now much more portable.

Thanks to [Patrick Fergus](https://foigus.wordpress.com) for testing and feedback with Enterprise licenses.

## You're welcome

Adobe, you could have made this so much easier. Please use this utility as an example of how you can make things at least a little bit easier.
