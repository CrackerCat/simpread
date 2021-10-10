> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [bohops.com](https://bohops.com/2021/10/08/analyzing-and-detecting-a-vmtools-persistence-technique/)

> Introduction It is always fun to reexplore previously discovered techniques or pick back on old resea......

Introduction
------------

It is always fun to reexplore previously discovered techniques or pick back on old research that was put on the wayside in hopes to maybe finding something new or different. Recently, I stood up an ESXi server at home and decided to take a quick peak at the VMware directory structure after installing the VMware Tools (vmtools) package in a Windows 10 Virtual Machine.

Among the directory contents were some batch files that I forgot about and the very interesting binary – VMwareToolBoxCmd.exe. After some quick Googling, it did not take take long to land on Adam’s ([@Hexacorn](https://twitter.com/Hexacorn)) incredible blog to find these two very informative post about VMwareToolBoxCmd.exe, OS fingerprinting, and a privileged persistence technique with VMware Tools:

*   [Beyond good](https://www.hexacorn.com/blog/2017/01/14/beyond-good-ol-run-key-part-53/) [ol](https://www.hexacorn.com/blog/2017/01/14/beyond-good-ol-run-key-part-53/)[’ Run key, Part 53](https://www.hexacorn.com/blog/2017/01/14/beyond-good-ol-run-key-part-53/)
*   [Using Virtual Machine tools for Guest OS fingerprinting](https://www.hexacorn.com/blog/2018/12/19/using-virtual-machine-tools-for-guest-os-fingerprinting/)

In this quick post, we will analyze this persistence technique and discuss a few strategies for detecting potential abuse.

The Technique
-------------

As Adam describes, VMwareToolBoxCmd.exe is a utility command for capturing VM information or changing the configuration of various and sundry virtual machine settings. One feature is to control batch scripts that can be configured to run based on VM state operations including _power_ (power on), _shutdown_ (power off), _resume_ (from suspended state), and and _suspend_ (entering suspended stated) as noted in the command utilities _script_ help subcommand:

![](https://bohops.files.wordpress.com/2021/10/image-1.png?w=808)

There are several built-in batch scripts in the VMware Tools directory, but this does not preclude someone from using and enabling a custom script. For example, the following command script can be used to specify the execution of a custom script when the VM is powered on:

```
VMwareToolboxCmd.exe script power set "c:\evil\evilscript.bat"
VMwareToolboxCmd.exe script power enable

```

The command sequence itself is not as interesting as what it actually does. In the following Sysmon screenshot, we can see that content is actually written to the _tools.conf_ file in \ProgramData:

![](https://bohops.files.wordpress.com/2021/10/image-2.png?w=808)

Upon further inspection, the contents of this file appear as follows:

![](https://bohops.files.wordpress.com/2021/10/image-4.png?w=808)

Coincidently, there is another operation for _resume_ under the _powerops_ section directive. This was added previously by me to show that 1) batch files are not the only thing that can be configured and 2) the _tools.conf_ file is the key component what enables the script execution functionality.

**_Note_**: For a complete example of what a configuration file may look like, take a look at _tools.conf.example_ in the same \ProgramData directory or this [sample](https://github.com/vmware/open-vm-tools/blob/master/open-vm-tools/tools.conf) file in VMware’s open-vm-tools repository.

After a quick shutdown and power-on, we can see our batch file payload (notepad.exe) is executed by cmd.exe as a child process of _vmtoolsd.exe_ under the context of NT AUTHORITY\SYSTEM:

![](https://bohops.files.wordpress.com/2021/10/image-5.png?w=808)

Defensive Considerations
------------------------

Consider the following detection opportunities:

**Sysmon**

For event collection with Sysmon, consider monitoring _tools.conf_ write (modification) events with an experimental rule. The following rule can be added to [@SwiftOnSecurity](https://twitter.com/SwiftOnSecurity)‘s [Sysmon-Config](https://github.com/SwiftOnSecurity/sysmon-config) under the _EVENT 11: “File created”_ section or under [@olafhartong](https://twitter.com/olafhartong)‘s [Sysmon-Modular](https://github.com/olafhartong/sysmon-modular) “11_file_create” rules:

```
<TargetFilename>C:\ProgramData\VMware\VMware Tools\tools.conf</TargetFilename>

```

**Elastic Security**

I’ve been digging into the Elastic Stack in recent months and felt that it would be a great opportunity to build a simple rule in Elastic Security. Conveniently, Elastic was kind enough to implement a rule creation wizard. Leverage this by selecting Elastic Security in Kibana, navigating to “Rules”, then selecting “Create New Rule”:

![](https://bohops.files.wordpress.com/2021/10/image-7.png?w=820)

I created this ‘custom’ rule based on the Event Query Language (EQL) of another [rule](https://github.com/elastic/detection-rules/blob/main/rules/windows/persistence_ms_office_addins_file.toml) [License: [Elastic License v2](https://github.com/elastic/detection-rules/blob/main/LICENSE.txt)]:

```
file where event.type != "deletion" and
file.path :
(
"C:\ProgramData\VMware\VMware Tools\tools.conf"
)

```

After walking through the wizard and enabling the rule, I modified the _tools.conf_ file which triggered this alert:

![](https://bohops.files.wordpress.com/2021/10/image-8.png?w=416)

Of note, the community can contribute to Elastic’s open-source Detection Rules [repository](https://github.com/elastic/detection-rules). There is a set of instructions to leverage a Python utility to help with the creation and validation process (outlined [here](https://github.com/elastic/detection-rules/blob/main/CONTRIBUTING.md)).

**Other Detection Opportunities**

***Environment**: In some environments, it is very plausable that operational power scripts/commands may already be enabled for legitimate reasons. If such is the case, audit the _tools.conf_ file for target scripts and monitor accordingly. Although custom scripts can be specified, the following (default) operational state scripts are included with VMware Tools (in the \VMware Tools directory) and may be worth monitoring:

*   poweroff-vm-default.bat
*   poweron-vm-default.bat
*   resume-vm-default.bat
*   suspend-vm-default.bat

***Hunt**: As shown in a previous screenshot, the parent process for the launched process is _vmtoolsd.exe_. Consider monitoring or hunting for suspicious child processes. Additionally, monitoring for VMwareToolBoxCmd.exe command usage could be opportunistic in some environments.

Conclusion
----------

As always, thank you for taking the time to read this post.

~ bohops