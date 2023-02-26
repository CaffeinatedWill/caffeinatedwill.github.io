---
layout: post
title: PowerShell Remoting from Linux to Windows
subtitle: How to connect to Windows from Linux via PowerShell
date: 2018-11-03T04:00:00.000+00:00
pageImageCard: "/images/powershell-remoting-from-linux-to-windows/demo.png"
content_img_path: ''
description: Solutions for those who face the Linux-to-Windows PowerShell remoting struggle
canonical_url: ''
tags:
- PowerShell
- Docker
- WinRM
- Linux
- PowerShell Remoting
- Linux-to-Windows
- Tool
published: true
quickbreach: true
redirect_from:
 - /blog/powershell-remoting-from-linux-to-windows/


---
\****Update \[10/06/2019\]:** As a result of this post, some of the official Microsoft PowerShell docker images were updated to support NTLM authentication ([See here for details](https://github.com/PowerShell/PowerShell-Docker/issues/124))

***

**TL;DR** You can PS-Remote from Linux to Windows if you

1. Permit NTLM authentication on a target during post-exploitation
2. Restart WinRM services
3. Use this NTLM supporting PowerShell Docker image to PS-Remote from Linux to Windows.

### Background Info

Occasionally I have found it useful on my pentests to leverage PowerShell remoting as my primary means of maintaining remote code execution on a system. It's a built-in Windows feature that is incredibly useful when living-off-the-land is a requirement due to client detection capabilities. Unfortunately, remoting from my Kali Linux box to my targets has not been an easy task due to the limited authentication mechanisms supported by the Linux branch of PowerShell Core.

PowerShell remoting requires Kerberos mutual authentication, which means that the client machine and target machine must both be connected to the domain. That can pose an issue for us testers if we don't already have a compromised domain-connected box to perform the remoting from. Fortunately, we have the option to add ourselves as a "TrustedHost" in the target's configuration which will permit us to perform NTLM authentication rather than Kerberos and thus removes the need to connect from a system on the domain.

Now the only catch is that PowerShell core for Linux (_PowerShell 6.1.0 at the time of writing_) does not come with support for NTLM authentication. This isn't the end of the world, as we could always perform PowerShell remoting from a Windows VM. Luckily some [Redditors](https://www.reddit.com/r/PowerShell/comments/6itek2/powershell_remoting_linux_windows_with_spnego/) found a way to get NTLM authentication working with PowerShell on Centos, and thus I incorporated their findings into the simple PowerShell Docker image [quickbreach/powershell-ntlm](https://hub.docker.com/r/quickbreach/powershell-ntlm/).

### How to use PowerShell remoting from Linux to Windows

This section will go through step-by-step how to establish a remote PowerShell session from a Linux client to a Windows target. It is assumed that you have administrative access over your target PC (RDP, payload, etc.).

Enable PowerShell remoting on the target by running the below command from an elevated PowerShell prompt

        Enable-PSRemoting –Force

Get a list of the current TrustedHosts on the target system for reference

        Get-Item WSMan:\localhost\Client\TrustedHosts

Next, add yourself as a TrustedHost on the target. This is required in order to use NTLM authentication during the Enter-PSSession setup phase, which is the only authentication mechanism that can be used to connect from Linux to Windows via PowerShell remoting. To accomplish this, run one of the below commands:

_Use a wildcard to permit all machines to use NTLM when authenticating to this host_

        Set-Item WSMan:\localhost\Client\TrustedHosts -Force -Value *

_Or be specific and add only your IP to the NTLM authentication permitted list_

        Set-Item WSMan:\localhost\Client\TrustedHosts -Force -Concatenate -Value 192.168.10.100

Setup and restart the WinRM service to reflect the changes made

``` 
Set-Service WinRM -StartMode Automatic
Restart-Service -Force WinRM
```

Now drop into an instance of the PowerShell-NTLM Docker image. The below sample command also mounts a local directory containing PowerShell scripts on the /mnt path inside of the docker image

    docker run -it -v /pathTo/PowerShellModules:/mnt quickbreach/powershell-ntlm

Finally, the moment we've been waiting for: Enter into a remote PowerShell session with the below commands - note that you MUST specify the -Authentication type:

       # Grab the creds we will be logging in with
    
       $creds = Get-Credential
    
       # Launch the session
    
       # Important: you MUST state the  authentication type as Negotiate
    
       Enter-PSSession -ComputerName (Target-IP) -Authentication Negotiate -Credential $creds
    
       # i.e.
    
       Enter-PSSession -ComputerName 10.20.30.190 -Authentication Negotiate -Credential $creds

You can also use the Invoke-Command function in a similar fashion

        Invoke-Command -ComputerName 10.20.30.190 -Authentication Negotiate -Credential $creds -ScriptBlock {Get-HotFix}

![](/assets/img/posts/01/demo.png "Pow")

### Clean up

If there were existing TrustedHosts previous to your command to add yourself, swap out your IP and run the below:

    $newvalue = ((Get-ChildItem WSMan:\localhost\Client\TrustedHosts).Value).Replace(",192.168.10.100","")
    
    Set-Item WSMan:\localhost\Client\TrustedHosts -Force -Value $newvalue

Or, you can remove all of the TrustedHosts if you were the only one

    Clear-Item WSMan:\localhost\Client\TrustedHosts

Restart the WinRM service to complete your changes (Note that this will disconnect you from Enter-PSSession)

    Restart-Service WinRM

### References

[This Reddit Thread](https://www.reddit.com/r/PowerShell/comments/6itek2/powershell_remoting_linux_windows_with_spnego/)  
[PowerShell Remoting Cheatsheet](https://blog.netspi.com/powershell-remoting-cheatsheet/)