---
layout: post
title:  "Reset a Forgotten Windows Administrator Password"
date:   2019-09-13 14:21:47 +1000
categories: Windows 
tags:
- System Administration
---

Assuming you don’t use a Microsoft account to log into Windows, you’ll have to reset the local password. If the locked account is the only administrator account on the PC, you need to first enable the hidden admin account to use this workaround.

The best option is to use the Microsoft tool [Windows PE](https://docs.microsoft.com/en-us/windows-hardware/manufacture/desktop/winpe-intro) to reset the password.

The tool allows you to do alot more as well :
* Set up your hard drive before installing Windows.
* Install Windows by using apps or scripts from a network or a local drive.
* Capture and apply Windows images.
* Modify the Windows operating system while it's not running.
* Set up automatic recovery tools.
* Recover data from unbootable devices.
* Add your own custom shell or GUI to automate these kinds of tasks.

## Create bootable USB device

If you don’t have one already, create Windows 10 installation media on a flash drive using another machine if necessary. 

* [Download and install the Windows ADK](https://docs.microsoft.com/en-au/windows-hardware/get-started/adk-install) 
* Install the Windows ADK
* Start the *Deployment and Imaging Tools Environment* tool (use start menu) as administrator.
* Extract the Windows PE tool with the command ```copype amd64 C:\WinPE_amd64```
* Create a bootable USB drive with WinPe ```MakeWinPEMedia /UFD C:\WinPE_amd64 P:``` where P: is your USB drive.


## Boot with the USB device

Insert that drive into your machine, and watch for a prompt to press F12, Delete, or another key to choose your boot device. Boot from the flash drive, and wait until you see the first Windows 10 setup screen.

You should now have a commadn prompt.

## Check disk device

Type ```diskpart``` and check that you can access the bootable volume :

```
DISKPART> LIST VOL

  Volume ###  Ltr  Label        Fs     Type        Size     Status     Info
  ----------  ---  -----------  -----  ----------  -------  ---------  --------
  Volume 0     C   Windows      NTFS   Partition    476 GB  Healthy    Boot
  Volume 1                      FAT32  Partition    500 MB  Healthy    System
```

In this part its C:.

## Hack the Password

Type this command to browse to the System32 folder:

```cd Windows\System32```

### Add Command Prompt to Windows Lock Screen

Now, you can use a trick to change one of the elements on the Windows lock screen. The Ease of Access menu collects accessibility options like the on-screen keyboard and dictation for users with disabilities. Using text commands, you can replace this icon with a shortcut to the Command Prompt. Enter these two lines one at a time to back up the shortcut and replace it:

```ren utilman.exe utilman.exe.bak```

```ren cmd.exe utilman.exe```

That’s it for now, so type this command to reboot as normal:

```wpeutil reboot```

Back at the normal sign-in screen, click the **Ease of Access** shortcut in the bottom-right to open a Command Prompt.

Type this command to enable the Admin account:

```net user Administrator /active:yes```

Now you need to reboot again. Use this command as a shortcut:

```shutdown -t 0 -r```

Once the system rebooted then click the **Ease of Access** shortcut in the bottom-right to open a Command Prompt.

## Reset the Password

List all users :

```net userl```

Your username should be obvious. Now, replace USERNAME in this command with yours, and Windows will let you set a new password:

```net user USERNAME *```

Then reboot

```shutdown -t 0 -r```

Insert that drive into your machine, and watch for a prompt to press F12, Delete, or another key to choose your boot device. Boot from the flash drive, and wait until you see the first Windows 10 setup screen.

You should now have a command prompt again.

## Revert the hack

Browse to ```C:\Windows\System32```, then type these two commands to fix the shortcut you changed:

```ren utilman.exe cmd.exe```

```ren utilman.exe.bak utilman.exe```

Done!

Remove the USB device and restart the system, you should be able to login using your local admin account.