---
layout: post
title: VMWare to Hyper-V
date: '2016-08-31 11:05:23'
---

Since I wanted to use Docker on Windows 10, I needed Hyper-V installed. Once Hyper-V is enabled in Windows 10, you no longer can boot a VMWare image. This is how you can convert your VMDK files to VHD files to be used in Hyper-V.

**Prep work**

Install the Powershell cmdlets included in [Microsoft's Virtual Machinve Convertor](https://www.microsoft.com/en-us/download/details.aspx?id=42497). There is a GUI that will be installed, but we want the PowerShell cmdlets that are included, namely "ConvertTo-MvmvVirtualHardDisk"

While using, I ran repeatedly into this error:

`ConvertTo-MvmcVirtualHardDisk: The entry 1 is not a supported disk database entry for the descriptor.`

Based on [this post](http://stackoverflow.com/questions/37481737/error-when-converting-vmware-virtual-disk-to-hyperv) at Stackoverflow, I edited my VMDK file and commented out the following line:

`ddb.toolsInstallType = "1"`

so that it looks like this

`#ddb.toolsInstallType = "1"`

**Using**

Run PowerShell with Administrator rights.

Import the module

`C:\WINDOWS\system32> Import-Module ‘C:\Program Files\Microsoft Virtual Machine Converter\MvmcCmdlet.psd1’`

In my case, my vmdk file pointed to multiple VMDK files, so I choose to run it with the DynamicHardDisk option:

`C:\WINDOWS\system32> ConvertTo-MvmcVirtualHardDisk -SourceLiteralPath 'C:\Users\luksi1\Vmware\D003395\Windows 7.vmdk' -VhdType DynamicHardDisk -VhdFormat Vhdx -DestinationLiteralPath C:\Users\luksi1\HyperV\D003395\`


