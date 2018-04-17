---
layout: post
title: How to disable Meltdown and Spectre Patches on Linux
date:   2018-04-17 17:00:00 +0800
categories: technology security
---

[Meltdown and Spectre][] are vulnerabilities in modern processors,
which allow a rogue process to read all memory which is currently processed on the computer, including passwords, documents, security credentials, and your photos. Your operating system and software may have included corresponding patches to mitigate those vulnerabilities if they are up to date.
However, those patches significantly slowdown your computer.

If you have offline computers that run only trusted software, you may want to disable those patches to regain performance lost. This article will show you how to disable those patches on Linux. [There is also an instruction for Windows][Windows Meltdown and Spectre].

### Check if Meltdown and Spectre patches are applied
Run the following command as root:
``` sh
grep . /sys/devices/system/cpu/vulnerabilities/*
```

The following output means that the patches are enabled for the Meltdown and Spectre vulnerability:

``` sh
/sys/devices/system/cpu/vulnerabilities/meltdown:Mitigation: PTI
/sys/devices/system/cpu/vulnerabilities/spectre_v1:Mitigation: __user pointer sanitization
/sys/devices/system/cpu/vulnerabilities/spectre_v2:Mitigation: Full generic retpoline, IBPB, IBRS_FW
```

### Disable vulnerability fixes
You can disable those patches by adding `nopti` boot option to kernel.
Assume your Linux distribution uses grub2 as the boot manager:

Firstly modify `GRUB_CMDLINE_LINUX` parameter in the `/etc/default/grub` and append `nopti` boot option at the end of the line, like:
``` sh
GRUB_CMDLINE_LINUX="rd.lvm.lv=fedora/root rd.lvm.lv=fedora/swap rhgb quiet nopti"
```

Then regenerate the grub config file.
If your OS is booted from legacy BIOS, run:
``` sh
grub2-mkconfig -o /etc/grub2.cfg
```
Otherwise if your OS is booted from UEFI, run:
``` sh
grub2-mkconfig -o /etc/grub2-efi.cfg
```

Lastly, reboot your computer.

### Verify if the patches are disabled
Rerun the following command as root:
``` sh
grep . /sys/devices/system/cpu/vulnerabilities/*
```

If successful, your output should be something like:
``` shspectre fedora
/sys/devices/system/cpu/vulnerabilities/meltdown:Vulnerable
/sys/devices/system/cpu/vulnerabilities/spectre_v1:Vulnerable
/sys/devices/system/cpu/vulnerabilities/spectre_v2:Mitigation: Full generic retpoline
```

### Re-enable the patches
Just remove the `nopti` boot option from `/etc/default/grub` and regenerate the grub boot config file like what we did before.

[Meltdown and Spectre]: https://meltdownattack.com/
[Windows Meltdown and Spectre]: https://social.technet.microsoft.com/Forums/windowsserver/en-US/fd9f2f4f-2534-4d61-86cd-fa5f38ac1557/meltdown-and-spectre-must-registry-value-featuresettingsoverride-manually-set-after-patch?forum=winserver8gen
