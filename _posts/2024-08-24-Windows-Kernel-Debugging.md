---
title: Windows Kernel Debugging
date: 2024-08-24 10:00:00 -0500
categories: [wke, pwn]
tags: [windows, kernel, pwn, wke]
author: oreo
dsecription: Windows Kernel Debugging
---

## Introduction

Kernel debugging isn't that much different than user-mode debugging from my experience, it just requires more setup and you can't (or shouldn't) do it locally. Here's a high-level overview of what you need to do:

1. Create a Windows VM - preferably hosted on Hyper-V
2. Install WinDBG on host computer and KDNET on VM
3. Enable debug mode and test-signing
4. Set-up KDNET
5. Attach WinDBG to VM kernel
6. Load & Run driver
    - Optional: Apply a mask to see appropriate `DbgPrint()` messages

## Getting WinDbg

Installing WinDBG will be the easiest part of this tutorial (if you are running Windows locally). Microsoft released an new and improved version of WinDBG (thankfully) that can be easily installed through the Microsoft Store.

All you need to do is:
1. Open Microsoft Store
2. In the search bar, enter "WinDbg"
3. Press install

It should look like this when you're done:

![WinDbg](/assets/images/WKE/WinDbg.png)

## Installing KDNET
{: .mt-4 .mb-0 }

This is where things start becoming a little tricky (not too bad though).

### Troubleshooting Tricks

If WinDBG is unable to connect to KDNET on your VM here are some things to try:
1. **ENABLE** ICMP (ping) traffic inbound on the firewall
    - By default, Windows blocks ICMP (ping) packets. Not sure why, but it does.
2. Verify you can ping the VM: `ping <VM IP>`
    - If you can't, it's probably one of two things:
        - Your VM isn't on the correct network - change the VM's network adapter
        - Ping is probably blocked by the VM's firewall (see [1] )

## Loading & Running the Driver

I will come back and flesh this out more, but basically you need to create a service and start the service from an Administrator command prompt. You will need to create a kernel service:

```dos
C:\> sc create [service name] binPath= C:\path\to\driver type= kernel
```
> This should say something along the lines of "SUCCESS"
{: .prompt-info}

Then to start the driver, run: 
```dos
C:\> sc start [service name]
```

If any errors occur when the service starts (mostly permissioning or FILE NOT FOUND), it will tell you. In other words - No output, is good output. If you're concerned about whether the driver is currently loaded/running, run this command: `sc query [service name]`.