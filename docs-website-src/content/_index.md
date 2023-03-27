---
title: "cPanel ELevate CloudLinux 7 to CloudLinux 8"
date: 2023-03-27T11:50:12+04:00
draft: false
layout: single
---

# Welcome to the cPanel ELevate Project - CloudLinux variant.

## Goal

The [cPanel ELevate Project](https://github.com/cpanel/elevate) provides a script to upgrade an existing `cPanel & WHM` [CentOS&nbsp;7](https://centos.org) server installation to [AlmaLinux&nbsp;8](https://almalinux.org) or [Rocky&nbsp;Linux&nbsp;8](https://rockylinux.org).

This repository contains a modification of said script that allows upgrading from CloudLinux 7 server installations to CloudLinux 8.

## Disclaimer

The functionality of software in this repository is not guaranteed.  We provide it on an experimental basis only. You assume all risk for use of any software that you install from this experimental repository. Installation of this software could cause significant functionality failures, even for experienced administrators.

## Introduction

- [Issues can be reported here](https://github.com/cloudlinux/elevate/issues)
- [Pull requests are welcome](https://github.com/cloudlinux/elevate/pulls)

This project builds on the [Alma Linux ELevate](https://wiki.almalinux.org/elevate/ELevate-quickstart-guide.html) project and its modification, [CloudLinux ELevate](https://docs.cloudlinux.com/elevate/), which lean heavily on the [LEAPP Project](https://leapp.readthedocs.io/en/latest/) created for in-place upgrades of RedHat-based systems.

The [Alma Linux ELevate](https://wiki.almalinux.org/elevate/ELevate-quickstart-guide.html) project is effective at upgrading the distro packages from [CentOS&nbsp;7](https://centos.org/) to [AlmaLinux&nbsp;8](https://almalinux.org/) or [Rocky&nbsp;Linux&nbsp;8](https://rockylinux.org). Its modification, [CloudLinux ELevate](https://docs.cloudlinux.com/elevate/) is also capable of upgrading CloudLinux 7 systems to CloudLinux 8.

However if you attempt use them directly on a CentOS 7 or CloudLinux 7-based [cPanel&nbsp;install](https://cpanel.net/), you will end up with a broken system.

This project was designed to be a wrapper around the **ELevate** project to allow you to successfully upgrade a [cPanel install](https://cpanel.net/) with an aim to minimize outages.

### Our current approach can be summarized as:

1. [Check for blockers](https://cpanel.github.io/elevate/blockers/)
2. `yum update && reboot`
3. Analyze and remove software (not data) commonly installed on a cPanel system
4. [Execute the Leapp upgrade](https://docs.cloudlinux.com/elevate/)
5. Re-install previously removed software detected prior to upgrade. This might include:
  * cPanel (upcp)
  * EA4
  * Distro Perl/PECL binary re-installs
6. Final reboot (assure all services are running on new binaries)

## Risks

As always, upgrades can lead to data loss or behavior changes that may leave you with a broken system.

Failure states include but are not limited to:

* Failure to upgrade the kernel due to custom drivers
* Incomplete upgrade of software because this code base is not aware of it.

We recommend you back up (and ideally snapshot) your system so it can be easily restored before continuing.

This upgrade will potentially take 30-90 minutes to upgrade all of the software. During most of this time, the server will be degraded and non-functional. We attempt to disable most of the software so that external systems will re-try later rather than fail in an unexpected way. However there are small windows where the unexpected failures leading to some data loss may occur.

## Before updating

Before updating, please check that you met all the pre requirements:

* You will need to have console access available to your machine
* You should back up your server before attempting this upgrade
* Ensure your server is up to date: `yum update`
* Ensure you are using the last stable version of cPanel & WHM

Additional checks can be performed by [downloading the script](#download-the-elevate-cpanel-script)
and then [running pre-checks](#pre-upgrade-checks).

### Some of the problems you might find include:

* EA4 RPMs are incorrect
  * EA4 provides different dependencies and linkage on C7/A8
* cPanel binaries (cpanelsync) are invalid.
* 3rd-party repo packages are not upgraded.
* Manually installed Perl XS (arch) CPAN installs invalid.
* Manually installed PECL need to be re-built.
* Cpanel::CachedCommand is wrong.
* Cpanel::OS distro setting is wrong.
* MySQL might now not be upgradable (MySQL versions < 8.0 are not normally present on A8).
* The `nobody` user does not switch from UID 99 to UID 65534 even after upgrading to A8.

## Using the script

### Download the elevate-cpanel script

* You can download a copy of the script to run on your cPanel server via:

```bash
wget -O /scripts/elevate-cpanel \
    https://raw.githubusercontent.com/cloudlinux/elevate/cloudlinux-release/elevate-cpanel ;
chmod 700 /scripts/elevate-cpanel
```

### Pre-upgrade checks

We recommend you check for known blockers before you upgrade. The check is designed to not make any changes to your system.

You can check if your system is ready to upgrade by running:
```bash
# Check upgrade eligibility (dry run mode)
/scripts/elevate-cpanel --check # defaults to CloudLinux if run on CloudLinux, AlmaLinux otherwise
```

### To upgrade

Once you have a backup of your server (**The cPanel elevate script does not back up before upgrading**), and have cleared upgrade blockers with Pre-upgrade checks, you can begin the migration.

**NOTE** This upgrade could take over 30 minutes. Be sure your users are aware that your server may be down and
unreachable during this time.


You can start the upgrade by running:
```bash
# Start the migration
/scripts/elevate-cpanel --start
```

CloudLinux 7 systems are automatically upgraded to CloudLinux 8.

### Command line options

```bash
# Read the help (and risks mentionned in this documentation)
/scripts/elevate-cpanel --help

# Check if your server is ready for elevation (dry run mode)
/scripts/elevate-cpanel --check # defaults to CloudLinux if run on CloudLinux

# Start the migration
/scripts/elevate-cpanel --start # defaults to CloudLinux if run on CloudLinux

... # expect multiple reboots (~30 min)

# Check the current status
/scripts/elevate-cpanel --status

# Monitor the elevation log
/scripts/elevate-cpanel --log

# In case of errors, once fixed, you can continue the migration process
/scripts/elevate-cpanel --continue
```

## Upgrade process overview

The elevate process is divided in multiple `stages`.
Each `stage` is responsible for one part of the upgrade.
Between stages, a `reboot` is performed, with one last reboot at the end of the final stage.

### Stage 1

Start the elevation process by installing the `elevate-cpanel` service responsible for controlling the upgrade process between multiple reboots.

### Stage 2

Update the current distro packages.
Disable cPanel services and setup the custom upgrade MOTD.

### Stage 3

Setup the Leapp ELevate package repository and install Leapp packages.
Prepare the cPanel packages for the update.

Remove some known conflicting packages and back up some existing configurations. These packages will be reinstalled later.

Provide answers to a few leapp questions.

Attempt to perform the `leapp` upgrade.

In case of failure you probably want to reply to a few extra questions or remove some conflicting packages.

### Stage 4

At this stage we should now run CloudLinux 8.
Update cPanel product for the new distro.

Restore the packages that were removed during the previous stage.

### Stage 5

This is the final stage of the upgrade process.
Perform some sanity checks and cleanup.
Remove the `elevate-cpanel` service used during the upgrade process.

A final reboot is performed at the end of this stage.

## FAQ

### How to check the current status?

You can check the current status of the elevation process by running:
```
/scripts/elevate-cpanel --status
```

### How to check elevate log?

The main log from the `/scripts/elevate-cpanel` can be read by running:
```
/scripts/elevate-cpanel --log
```

### Where to find leapp issues?

If you need more details why the leapp process failed you can access logs at:
```
        /var/log/leapp/leapp-report.txt
        /var/log/leapp/leapp-report.json
```

### How to continue the elevation process?

After addressing the reported issues, you can continue an existing elevation process by running:
```
/scripts/elevate-cpanel --continue
```

### I need more help?

You can report an issue to the [Github Issues page](https://github.com/cloudlinux/elevate/issues)

## Copyright

```c
Copyright 2023 cPanel L.L.C.

Redistribution and use in source and binary forms, with or without modification,
are permitted provided that the following conditions are met:

1. Redistributions of source code must retain the above copyright notice,
   this list of conditions and the following disclaimer.
2. Redistributions in binary form must reproduce the above copyright notice,
   this list of conditions and the following disclaimer in the documentation
   and/or other materials provided with the distribution.

THIS SOFTWARE IS PROVIDED BY THE COPYRIGHT HOLDERS AND CONTRIBUTORS "AS IS"
AND ANY EXPRESS OR IMPLIED WARRANTIES, INCLUDING, BUT NOT LIMITED TO,
THE IMPLIED WARRANTIES OF MERCHANTABILITY AND FITNESS FOR A PARTICULAR PURPOSE
ARE DISCLAIMED. IN NO EVENT SHALL THE COPYRIGHT HOLDER OR CONTRIBUTORS BE LIABLE
FOR ANY DIRECT, INDIRECT, INCIDENTAL, SPECIAL, EXEMPLARY, OR CONSEQUENTIAL DAMAGES
(INCLUDING, BUT NOT LIMITED TO, PROCUREMENT OF SUBSTITUTE GOODS OR SERVICES;
LOSS OF USE, DATA, OR PROFITS; OR BUSINESS INTERRUPTION) HOWEVER CAUSED
AND ON ANY THEORY OF LIABILITY, WHETHER IN CONTRACT, STRICT LIABILITY, OR TORT
(INCLUDING NEGLIGENCE OR OTHERWISE) ARISING IN ANY WAY OUT OF THE USE OF THIS SOFTWARE,
EVEN IF ADVISED OF THE POSSIBILITY OF SUCH DAMAGE.
```
