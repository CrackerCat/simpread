> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [snyk.io](https://snyk.io/blog/abusing-ubuntu-root-privilege-escalation/)

> With the recent release of Ubuntu 24.04, we at Snyk Security Labs thought it would be interesting to ......

Introduction
------------

With the recent release of Ubuntu 24.04, we at Snyk Security Labs thought it would be interesting to examine the latest version of this Linux distribution to see if we could find any interesting privilege escalation vulnerabilities.

I’ll let the results speak for themselves:

```
1rory@ubuntu2404release:~$ cat /etc/lsb-release
2DISTRIB_ID=Ubuntu
3DISTRIB_RELEASE=24.04
4DISTRIB_CODENAME=noble
5DISTRIB_DESCRIPTION="Ubuntu 24.04 LTS"
6rory@ubuntu2404release:~$ id
7uid=1000(rory) gid=1000(rory) groups=1000(rory),4(adm),24(cdrom),27(sudo),30(dip),46(plugdev),100(users),114(lpadmin)
8rory@ubuntu2404release:~$ sh exploit.sh
9Re-executing via Desktop
10Disabling notifications
11Staging cups symlinks
12Taking over cupsd.conf
13Rewriting cupsd.conf
14Taking over cups-files.conf
15Setting filter group to netdev
16Staging for wpa_supplicant
17Triggering wpa_supplicant as group netdev via cups
18Re-enabling notifications
19Cleaning up and providing shell
20# id
21uid=0(root) gid=0(root) groups=0(root)
```

**During our research**, **we successfully identified a privilege escalation from the default user on a fresh Ubuntu Desktop installation to root.** To achieve this, we chained one small bug in a privileged component together with a number of features, which all work as expected, to achieve arbitrary command execution as root.

These vulnerabilities were promptly fixed by the respective maintainers and patches are generally available as of the 6th of August 2024. For a more detailed timeline please see the bottom of this post.

This blog post will outline the journey of our research, discuss how we identified these vulnerabilities, and, we hope, show that you can keep it simple when it comes to exploitation and achieve the same results without needing a very complex (although extremely cool) kernel memory corruption vulnerability, for example.

Hunting for root privilege escalation on Linux
----------------------------------------------

### User privilege boundaries

Whenever I’m looking at something new, especially for privilege escalation vulnerabilities, my first thought is always, “Where are the privilege boundaries?”. This is not a novel place to start, but I find it is always very useful to define what specific thing it is I’m trying to break. There’s no use going into a product to look for vulnerabilities with no plan for what they will even look like.

When looking at a Linux system such as Ubuntu, there are a handful of very common privilege boundaries. Some, such as setuid binaries, bad file permissions, and Unix sockets, have been around for a long time. I generally skip these things on first blush — they’ve often already been closely examined. I do, of course, come back to them later, but for the density of vulnerabilities, my first port of call is always DBus.

DBus is, for our purposes at least, a client/server-based remote procedure call (RPC) framework. What this means in practice is that a server, for example, a privileged component that manages users on the system, can register itself on the DBus message bus and offer up a number of methods, which can then be called by clients on the same bus. There are a number of built-in security controls for this bus, including explicit rules for which users or groups can or cannot call methods and even which users can claim to be a specific service. We will expand deeper into the permission model later on.

I am heavily simplifying what DBus is and can do here, but for the sake of brevity, let’s define DBus as “a service which allows certain users to call methods inside another, possibly privileged, process,” which gives us enough context to establish our privilege boundary of interest.

### Investigating DBus

[Many](https://unit42.paloaltonetworks.com/usbcreator-d-bus-privilege-escalation-in-ubuntu-desktop/) before me, and I’m sure many after me, have found vulnerabilities using the DBus interface. At a high level, tooling such as d-feet (now superseded by [D-Spy](https://apps.gnome.org/en-GB/Dspy/)) can allow you to enumerate all services that have registered methods on the DBus bus and let you test each of the registered methods (modulo permission checks) by showing you the correct arguments that should be provided.

I used this method to go through all of the services present on a new installation of Ubuntu 24.04 and review what may present an opportunity. After a few days and a fair few false starts, I eventually found the interface `org.opensuse.cupspkhelper`, which, by looking at the available methods, is used to manage the `cups` daemon (a service that manages printers and printing). Cups is a privileged process and I know from previous experience (repeatedly [exploiting](https://issues.chromium.org/issues/40092465) [cups](https://issues.chromium.org/issues/40052058) on Google’s ChromeOS) that if the privilege boundaries surrounding cups are not careful, cups can be used as a very powerful exploitation gadget.

![](https://snyk.io/_next/image/?url=https%3A%2F%2Fres.cloudinary.com%2Fsnyk%2Fimage%2Fupload%2Fv1725899741%2Fblog-ubuntu-org.png&w=2560&q=75)

_D-feet: Viewing the available methods in the org.opensuse.CupsPkHelper DBus interface_

With this in mind, a DBus interface that allows you to manage this daemon could result in some very interesting and unexpected behavior.

### This feels privileged

As you may know or have inferred, printer management tends to be an “admin” task and should generally be privileged. Why, then, are we able to call these methods without entering our password?

Enter Polkit. Gone are the days of needing to use sudo for everything, with its associated lack of granularity. With Polkit, every privileged action you may want to take can be given its own unique name, and authorization can be defined per-name and take into account how the action is being requested.

For example, if you want to change the system time, there’s a unique name for that: `org.freedesktop.timedate1.set-time`:

Following is the `/usr/share/polkit-1/actions/org.freedesktop.timedate1.policy` file contents:

```
1        <action id="org.freedesktop.timedate1.set-time">
2                <description gettext-domain="systemd">Set system time</description>
3                <message gettext-domain="systemd">Authentication is required to set the system time.</message>
4                <defaults>
5                        <allow_any>auth_admin_keep</allow_any>
6                        <allow_inactive>auth_admin_keep</allow_inactive>
7                        <allow_active>auth_admin_keep</allow_active>
8                </defaults>
9                <annotate key="org.freedesktop.policykit.imply">org.freedesktop.timedate1.set-timezone org.freedesktop.timedate1.set-ntp</annotate>
10        </action>
```

This XML for Polkit defines the actions that could be taken, what message should pop up when a password is requested (“Authentication is required to set the system time.”), and what the rules are around who can authenticate (`<allow_any>auth_admin_keep</allow_any>`). The `auth_admin_keep` means an administrator (i.e, someone in the “sudo” group) can use their password to change the time, and the `_keep` means that the authentication will be remembered for a short time, so you can change the time twice quickly without needing to re-enter your password.

But how does this apply to us? We aren’t prompted to enter our password when we call the methods on our target DBus interface. Looking at the policy definitions for `org.opensuse.cupspkhelper.mechanism`, we see that all the actions require `auth_admin`, which should need the admin password every time.

The solution to our puzzle is the second half of what makes Polkit powerful: rules. Polkit allows for the definition of rules (written in JavaScript), which can be used to change the behavior of what is defined in the policy file. Looking at the rule sets in the file `/usr/share/polkit-1/rules.d`, we can find the following snippet:

The contents of the file `/usr/share/polkit-1/rules.d/com.ubuntu.desktop.rules`:

```
1// Printer administration
2polkit.addRule(function(action, subject) {
3    if (action.id.indexOf("org.opensuse.cupspkhelper.mechanism.") == 0 &&
4        subject.active == true && subject.local == true &&
5        (subject.isInGroup("sudo") || subject.isInGroup("lpadmin"))) {
6            return polkit.Result.YES;
7    }
8});
```

This rule is made up of 3 main parts.

*   The first part, `action.id.indexOf("org.opensuse.cupspkhelper.mechanism.") == 0`, checks to see if this is one of the actions used by our DBus interface by checking the prefix. The specific rule looks similar to `org.opensuse.cupspkhelper.mechanism.printer-local-edit`.
    
*   The second section is `subject.active == true && subject.local == true`. This is Polkit speak for “Is the user logged in to the computer directly and sat looking at the screen?”. This applies to most direct uses of Ubuntu Desktop, but won’t apply for, SSH sessions on the same machine, for example.
    
*   The final section `(subject.isInGroup("sudo") || subject.isInGroup("lpadmin"))` checks to see if the requesting user is in either the “sudo” or “lpadmin” user groups.
    

If all 3 sets of conditions for this rule are found to be true, i.e a user who is sat in front of their computer, in the sudo or lpadmin groups, and trying to make changes to printers, this rule instructs the authorization check to succeed (`polkit.Result.YES`) without needing the user to enter their password. This is quite a nice usability feature and will decrease the amount of passwords an administrator will need to type in all the time.

The problems begin when an attacker is able to compromise the user’s session, potentially through social engineering to execute a payload, or other exploitation (_I_ may not have Chrome full RCE vulnerabilities, but they’ve happened before). In this context they will be able to call interfaces secured in this way without needing to first compromise the user’s password, which may have been necessary if these rules did not exist.

So we’ve found an interface which crosses a potentially interesting privilege boundary, and we’ve also found that we may be able to call these privileged functions without necessarily needing to compromise the user’s password. This is quite a good starting point for a privilege escalation chain, as privileged functions which are nominally protected, but can be accessed in certain circumstances bypassing said protection may allow us to exploit some interesting behavior.

Finding the bug
---------------

If you’ve read some of my [previous work](https://snyk.io/blog/leaky-vessels-container-vuln-deep-dive/) you may know that I love the `strace` tool. If you’re not familiar with this tool I encourage you to go back and read the first part of my [Leaky Vessels deep dive post](https://snyk.io/blog/leaky-vessels-container-vuln-deep-dive/), which contains a primer.

Whenever exploring an interface across a privilege boundary, it’s a very good habit to attach strace to your target and save the logs. This is a great way to see what’s _actually_ being done by the application when you call each method, and may give you quick wins from low hanging fruit.

Using this methodology, I observed that the cups daemon was restarting when calling `ServerSetSettings`. This was identified during the evaluation of the `ServerGetSettings` and `ServerSetSettings` methods. These functions are defined as follows (this output is taken from the `gdbus introspect` tool):

```
1      @org.freedesktop.DBus.GLib.Async("")
2      ServerGetSettings(out s error,
3                        out a{ss} settings);
4      @org.freedesktop.DBus.GLib.Async("")
5      ServerSetSettings(in  a{ss} settings,
6                        out s error);
```

For those not familiar with DBus datatypes, `a{ss}` is an “array (`a`) of structs (`{}`) mapping a string (`s`) to a string (`s`)”. This is functionally equivalent to a dictionary in Python where the keys and values are both strings. By calling ServerGetSettings to get a valid “settings” value and passing that same value to ServerSetSettings, cups performed a restart. From this it can be inferred that the settings being changed are those in cups itself, and the service is restarted to make the changes take effect.

This matches with what we can observe in the actual output of `ServerGetSettings`:

```
1{'BrowseLocalProtocols': 'dnssd',
2 'DefaultAuthType': 'Basic',
3 'ErrorPolicy': 'retry-job',
4 'IdleExitTimeout': '60',
5 'JobPrivateAccess': 'default',
6 'JobPrivateValues': 'default',
7 'MaxLogSize': '0',
8 'PageLogFormat': '',
9 'SubscriptionPrivateAccess': 'default',
10 'SubscriptionPrivateValues': 'default',
11 'WebInterface': 'Yes',
12 '_debug_logging': '0',
13 '_remote_admin': '0',
14 '_remote_any': '0',
15 '_share_printers': '0',
16 '_user_cancel_any': '0'}
```

Most of these values (those not starting with an underscore) match what can be seen in the cups configuration file `/etc/cups/cupsd.conf`. By changing some of these values and observing the contents of the cups configuration file we can confirm that we are successfully changing these items, and they are taking effect. Great! This is a privileged action and a privileged configuration file, by default a normal user can’t even read the file so it must be juicy, right?

A quick spin around the [man page](https://www.cups.org/doc/man-cupsd.conf.html) doesn’t turn up anything interesting unfortunately. Nothing so easy as “please run this command as root when you start”. However, we still have some options. The next step is to look for what our strace has turned up during this process. My first instincts are always to look for potentially unsafe handling of configuration values that we’ve provided, or other sections of the code that we can influence. With this in mind we can do some quick greps for key syscalls like `chmod`, `chown`, and `execve` to investigate further behavior.

Generally we’re looking for things that could potentially be influenced by the configuration file we have partial control over. After searching we can come across this strace snippet:

```
1unlink("/var/run/cups/cups.sock")       = -1 ENOENT (No such file or directory)
2umask(000)                              = 022
3bind(8<UNIX-STREAM:[120796]>, {sa_family=AF_UNIX, sun_path="/var/run/cups/cups.sock"}, 26) = 0
4umask(022)                              = 000
5chmod("/var/run/cups/cups.sock", 0140777) = 0
```

We know from when we validated our previous assumptions against the `cupsd` configuration file that `/var/run/cups/cups.sock` is defined as a “Listen” argument. So, we can potentially control the value used in these syscalls here. My initial reaction is that there is a race condition here. The `chmod` call does not happen against a file descriptor, so there is a small window between the bind call and the `chmod` call where the socket could be swapped out for a symbolic link, allowing for arbitrary `chmod`. A `chmod` of 0140777 (essentially the same as `chmod 777`) will set the target of the call to be readable and writable by all users on the system. Since the `cupsd` process runs as root, this is potentially a very useful exploitation gadget, as we could (for example) set `/etc/passwd` to be world-writable and change our user id to 0, effectively making us root!

If we look up this specific functionality in cups we can see the following snippet from `cups/cups/http-addr.c`:

```
1    // Remove any existing domain socket file...
2    unlink(addr->un.sun_path);
4    // Save the current umask and set it to 0 so that all users can access
5    // the domain socket...
6    mask = umask(0);
8    // Bind the domain socket...
9    status = bind(fd, (struct sockaddr *)addr, (socklen_t)httpAddrLength(addr));
11    // Restore the umask and fix permissions...
12    umask(mask);
13    chmod(addr->un.sun_path, 0140777);
```

Notice that the return values to neither unlink nor bind are checked until _after_ the `chmod` call is executed (omitted, `status` is checked later). The `bind` system call doesn’t like symlinks, so if there is one present at the provided path, the bind will fail. My assumption, based on previous experience and the `strace` output, was that the bind call return value would be checked, and if there was a symlink present, `cupsd` would quit with an error. But this was not the case in the code, so we could have our malicious symlink in place well before the call to bind and still get to the chmod call. This changes our initial assumption of a race condition into a vulnerability caused by not checking return values.

Eagle-eyed readers may also question the call to unlink and ask why that wouldn’t still make this a race condition. Whilst this may look like a major hurdle (and a return of the race condition), we will see shortly why this is not the case.

So, where are we now? We have found a method on the DBus interface which allows us to set arguments in the `cupsd.conf` configuration file, cups is then restarted and our configuration should take effect. We’ve also found some potentially unsafe handling of the Listen argument. We know that this is not a race condition.

So we can call our identified method, set Listen to a symlink to something interesting, and have write access to whatever we like now? To test this, we’ll create a new directory in `/tmp`, and add a symlink to `/etc/passwd` as our sample target. We can then call the DBus method and set Listen to this path (`/tmp/stage/passwd`). Unfortunately, this does not work:

```
1unlink("/tmp/stage/passwd")             = -1 EACCES (Permission denied)
2umask(000)                              = 022
3bind(6<UNIX-STREAM:[125588]>, {sa_family=AF_UNIX, sun_path="/tmp/stage/passwd"}, 20) = -1 EADDRINUSE (Address already in use)
4umask(022)                              = 000
5chmod("/tmp/stage/passwd", 0140777)     = -1 EACCES (Permission denied)
```

This is unexpected. `cupsd` is running as root, and it even has `CAP_DAC_OVERRIDE`. Why can’t we remove a file (the `unlink` system call) owned by a normal user, let alone run `chmod` for our own file? (`/etc/passwd` is owned by root)

Linux AppArmor
--------------

The answer lies in AppArmor.

### What is AppArmor

AppArmor is a Linux kernel security module that can restrict contained programs to only a specific set of resources. Similar to SELinux, AppArmor can be used to define profiles which an application will be executed with, and this profile specifies an allow and deny list of resources which the application can interact with.

A profile looks something like the following excerpt, taken from the `tcpdump` profile on my Ubuntu test machine:

```
1profile tcpdump /usr/bin/tcpdump {
2[...]
3  # for -z
4  /{usr/,}bin/gzip ixr,
5  /{usr/,}bin/bzip2 ixr,
7  # for -F and -w
8  audit deny @{HOME}/.* mrwkl,
9  audit deny @{HOME}/.*/ rw,
10  audit deny @{HOME}/.*/** mrwkl,
11  audit deny @{HOME}/bin/ rw,
12  audit deny @{HOME}/bin/** mrwkl,
13  owner @{HOME}/ r,
14  owner @{HOME}/** rw,
16  # for -r, -F and -w
17  /**.[pP][cC][aA][pP] rw,
18  /**.[pP][cC][aA][pP][nN][gG] rw,
19  /**.[cC][aA][pP] rw,
20[...]
21}
```

This profile will restrict what files any `tcpdump` process can access. For example, we can see that it is only allowed to execute (the `ixr` flag) a certain list of binaries (`gzip`, `bzip2`). We can also see that it is not allowed to access hidden files and directories in any user’s home directory but is explicitly allowed to write to directories owned by the user under which `tcpdump` is executing. Finally, we can see that the process is only allowed to write files outside of the home directory if the file extension is `.pcap` or similar. We can see this in action in the following snippet. Observe that, even though `/dev/shm` is world writable, `tcpdump` cannot write to this directory unless the file extension ends with a `pcap` extension.

```
1$ ls -ld /dev/shm
2drwxrwxrwt 2 root root 40 May 10 13:59 /dev/shm
3$ sudo tcpdump -i lo -w /dev/shm/test
4tcpdump: /dev/shm/test: Permission denied
5$ sudo tcpdump -i lo -w /dev/shm/test.pcap
6tcpdump: listening on lo, link-type EN10MB (Ethernet), snapshot length 262144 bytes
```

### Linux AppArmor, savior, and saboteur

Now that we’ve had a quick introduction to AppArmor, how does it apply to our cups vulnerability? We can see in `/etc/apparmor.d/usr.sbin.cupsd` that `cupsd` has its own profile.

There are two parts that we can attribute to the behavior of the strace output we observed earlier. Firstly, we can see that the policy imports `abstractions/user-tmp`. The abstractions folder contains a number of partial policies which are generic and can be reused. In this file, we can see the following snippet:

```
1  owner /tmp/**         rwkl,
2  /tmp/                 rw,
```

What this policy means is that `cupsd` has read and write access to the directory path at `/tmp` but only has read and write (and `lock` and `link`) access to subdirectories of `/tmp` _if it is the owner of the subdirectory_. Since our testing directory `/tmp/stage` is owned by my user and not root, `cupsd` is not allowed to make changes to this directory — so, the system call to `unlink` fails. This is a very convenient case of a security control actually _helping_ with exploitation, as without this restriction, the code we’ve identified as vulnerable would actually be a race condition as discussed above, but as it stands, because the `unlink` fails, there is no timing element.

The second part of the policy impacting our proof of concept is the lack of write access to `/etc`. This is why our `chmod /etc/passwd` attempt (via the symlink) is failing. This makes a lot of sense, because there’s no need for cups to be able to write to `/etc`, so why give it the opportunity?

We now have established why our proof of concept has failed. But all is not lost. We can use the AppArmor policy as an instruction manual for things that we know we _could_ successfully chmod and gain write access to. By reading through the policy, we can see that most of the places we have write access to are specific to the cupsd functionality, such as the following snippet, giving us full access to everything under `/etc/cups`.

```
1  /etc/cups/ rw,
2  /etc/cups/** rw,
```

Pivot, pivooot
--------------

We’ve gone from what looked like a very easy-to-exploit vulnerability to something that has quite a limited impact. Looking at the AppArmor policy, we can only really make changes to cups configuration, cache, or log files. We will somehow need to find a method to turn arbitrary control over all of the cups configuration files into something more interesting, our aim being full root command execution.

To decide what we can achieve with what we have so far, the first port of call is the documentation for the files we think we can take over. This will include `cupsd.conf`, `cups-files.conf`, and a handful of other files in `/etc/cups`. Reading through the documentation for files we come across two interesting snippets from the [cups-files.conf](https://www.cups.org/doc/man-cups-files.conf.html) man page:

```
1Group group-name-or-number
2        Specifies the group name or ID that will be used when executing external programs.         The default group is operating system specific but is usually "lp" or "nobody".
4User username
5        Specifies the user name or ID that is used when running external programs. The       default is "lp".
```

This sounds very interesting, as it allows for the specification of a user used for external programs. To actually try this out, however, we do need to exploit our vulnerability above to have a successful chmod against `/etc/cups/cups-files.conf`. There is a bit of hidden complexity in exploitation that hasn’t come up yet. When using the DBus `ServerSetSettings` method to change the Listen parameter, cups will no longer run properly. It will successfully execute the `chmod` call as we saw above, but because the bind call is unsuccessful, when the bind return value is checked later in the code cups will identify that the bind failed and exit. In this state, we’re not able to make any more changes, and, critically, we’re not able to exploit the vulnerability again.

To mitigate this, we can target `/etc/cups/cupsd.conf` _first_. Our attack will progress, this file will become world writable, and we can fix our inserted malicious Listen lines to be like the original lines — in essence turning our somewhat limited control of this file via DBus into full control. This will allow `cupsd` to successfully start again*, but `cupsd.conf` will remain world writable. Once in this state, we can add more Listen lines to the end of `cupsd.conf` (as `cupsd` will accept multiple of these lines) and trigger a restart of the cupsd service (by using `ServerSetSettings` on an unrelated and safe configuration item). We can now safely re-exploit the vulnerability whenever we like without losing access to `cupsd`, or breaking the `org.opensuse.CupsPkHelper` interface (which requires cups to be functional and listening at an expected location).

Using this we can then make `/etc/cups/cups-files.conf` world writable, and now we can add our own ‘User’ and ‘Group’ configuration lines we identified in the configuration documentation.

_*Starting stopped services is generally a privileged action. However,_ `_cupsd_` _is a special case because it’s configured to be “socket activated” by default, so by attempting to communicate with_ `_cupsd_` _where systemd thinks the socket is (our interference notwithstanding), systemd will happily start_ `_cupsd_` _for us, without the need for any extra privileges._

Be who you wanna be
-------------------

We have identified that we can set the User and Group for “external programs”. Naively, we would like to set the `User` and `Group` to root, giving us root access immediately (when we can control “external programs”). However, if we try to set the User/Group lines to this in the configuration file and trigger a restart of cups, we’ll get the following error.

From the source code of `cups/scheduler/conf.c`:

```
1	  if (!p->pw_uid)
2	  {
3	    cupsdLogMessage(CUPSD_LOG_ERROR,
4	                    "Will not use User %s (UID=0) as specified on line "
5			    "%d of %s for security reasons.  You must use a "
6			    "non-privileged account instead.",
7	                    value, linenum, CupsFilesFile);
8	    if (FatalErrors & CUPSD_FATAL_CONFIG)
9	      return (0);
10	  }
```

This error tells us that root (or any user with a `UID` of 0) is explicitly disallowed. Since this step does not immediately result in root command execution, we must find another user to target, which can act as a stepping stone to our target of full root.

Before we evaluate our attack surface further, we will take a brief moment to discuss “external programs”.

### Cups external programs

Cups is a complex beast — more complex than can be covered in an article of even this length. The following explainer is _heavily_ condensed (and likely overly simplified), mainly to cover the areas we care about. I did not find this technique for this specific chain of vulnerabilities. However, like many things in this post, it was identified by reading documentation.

Cups is a printing system for Unix (literally. The name CUPS stands for Common Unix Printing System). You can register printers with cups, defining their capabilities using a PPD (Postscript Printer Description) configuration file. This file describes common things like the paper size, duplex options, resolution, etc. This file can also be used to define the supported filetypes of the printer. There is no use sending a raw PDF file to a printer that cannot understand the format. Therefore this file instructs cups what to do when a specific filetype is printed.

Cups distributes a set of “filters”, which can be used to convert between these different standard formats. For example, the `pdftops` filter can convert a `pdf` file to a `ps` (postscript) file, so you can effectively print PDF files on printers that do not understand them by transparently converting them inside cups.

One of the filters that a PPD file can specify is “foomatic-rip”, which claims to be a “universal filter”. As part of the configuration for this filter, there is a very juicy option: `FoomaticRIPCommandLine`. Even the naming of this option may pique your interest. This option can be used to define a command line that will be executed when the filter is triggered.

I will skip over the tedious validation using strace (`-fe execve` is a useful flag if you wish to try this at home) and conclude the following.

First we create a PPD file for a fake printer, which contains a `FoomaticRIPCommandLine` item, itself containing a command line controlled by us. We then register a new printer using this PPD file. Helpfully, our previously identified `org.opensuse.CupsPkHelper` DBus interface also includes a method for adding a printer with a manually specified PPD file. With our printer now created, we can attempt to print to it with a sample PDF file. Observing with strace we would see our command line configured in the PPD file executing when the print job is submitted.

The end result of this is command injection, and because of our previous exploitation controlling the User and Group parameters for _exactly these external programs_, the programs will run as any User/Group we desire (with the previously discussed exception of root).

Our final step is to identify a way of getting from any user/group to root.

More vulnerability gadgets features
-----------------------------------

There are probably a lot of ways of achieving this. For example, `/dev/sda2` (my root disk) has a group of `disk`, and permissions `660`, so I could use the configuration to set my Group to disk and modify this file. However, when I am in this kind of situation, I always try to err on the side of simplicity. A lot of things can go wrong with modifying a raw disk (even with good tools like `debugfs`), and it can make exploitation fiddly and unreliable (I’ve done it [before](https://issuetracker.google.com/issues/172219872)). Therefore, I took the time to attempt to identify other options which may be more reliable or at least simpler. When reporting vulnerabilities to vendors, simplicity is always beneficial.

Much like the start of this post, I began by investigating what was available on the DBus bus. This time, however, I first looked at the permission settings in the DBus daemon (defined in `/usr/share/dbus-1/system.d`) to find services accessible by specific users or groups and not to my original user.

These configuration files look something like the following `/usr/share/dbus-1/system.d/wpa_supplicant.conf`:

```
1 <!DOCTYPE busconfig PUBLIC
2 "-//freedesktop//DTD D-BUS Bus Configuration 1.0//EN"
3 "http://www.freedesktop.org/standards/dbus/1.0/busconfig.dtd">
4<busconfig>
5        <policy user="root">
6                <allow own="fi.w1.wpa_supplicant1"/>
7                <allow send_destination="fi.w1.wpa_supplicant1"/>
8                <allow send_interface="fi.w1.wpa_supplicant1"/>
9                <allow receive_sender="fi.w1.wpa_supplicant1" receive_type="signal"/>
10        </policy>
11        <policy group="netdev">
12                <allow send_destination="fi.w1.wpa_supplicant1"/>
13                <allow send_interface="fi.w1.wpa_supplicant1"/>
14                <allow receive_sender="fi.w1.wpa_supplicant1" receive_type="signal"/>
15        </policy>
16        <policy context="default">
17                <deny own="fi.w1.wpa_supplicant1"/>
18                <deny send_destination="fi.w1.wpa_supplicant1"/>
19                <deny receive_sender="fi.w1.wpa_supplicant1" receive_type="signal"/>
20        </policy>
21</busconfig>
```

In this example, for `wpa_supplicant`, normal users (i.e. `context=”default”`) cannot `send_destination` (call) methods on the interface at all. User root (`user=”root”`) can own and call the methods, this is pretty standard but doesn’t help us because we can’t become root. However, the potentially interesting part here is group netdev being allowed to call methods on this interface as well. It turns out the ability for the netdev group to call the interface is actually a Debian-family-specific [addition](https://git.launchpad.net/ubuntu/+source/wpa/tree/debian/patches/02_dbus_group_policy.patch).

Here we can iterate over finding an interesting interface callable by a user or group we can achieve, enumerate the accessible functions, and read the documentation. I will skip that this time for brevity (and because I did not discover this feature during this research) and we can go straight on to what the gadget is.

The `wpa_supplicant` service runs as root and, as we saw in the above example, has an interface that could be callable with our previous exploitation by becoming group `netdev`. On this interface there is a method `CreateInterface`. This method is generally used to configure a network interface for wireless network security (e.g. WPA/802.1X). This method takes an argument `ConfigFile`, which is a path to a `wpa_supplicant.conf` configuration file. Looking at the [documentation example](https://w1.fi/cgit/hostap/tree/wpa_supplicant/wpa_supplicant.conf#n177) for this file shows a couple of options that can be used to load arbitrary shared objects which can provide various functionality to the supplicant (e.g. crypto routines).

We can, therefore, write a shared object with some payload inside it, create a configuration file which includes a reference to this shared object to be loaded, exploit our previous vulnerability chain to attain command execution with a group ID of netdev, call the `wpa_supplicant`’s `CreateInterface` DBus method with our configuration file, and achieve arbitrary code execution when `wpa_supplicant` loads the shared object. Because, as previously mentioned, `wpa_supplicant` runs as root, **we will now have arbitrary root code execution**.

In the actual proof of concept, we use the shared object to set the Python binary to be setuid root. This makes it very easy to finish our script with an interactive root shell, and perform the cleanup activities necessary to undo what we have changed, such as the cups file permissions.

Proof of concept wrap-up
------------------------

The following section combines everything we’ve learned into a walkthrough of what the proof of concept actually looks like, to achieve full root command execution.

1.  Create a staging area in `/tmp/stage` for the symbolic links which we will target with cups.
    
2.  Call the `ServerSetSettings` DBus method to change the cups `Listen` parameter to `/tmp/stage/cupsd.conf`, which is a symlink to `/etc/cups/cupsd.conf`.
    
3.  The file `/etc/cups/cupsd.conf` will now be world-writable. Fix up the changes made by `ServerSetSettings` and add an additional `Listen` configuration item pointing to `/tmp/stage/cups-files.conf`, which is a symlink to `/etc/cups/cups-files.conf`.
    
4.  Use `ServerSetSettings` to change the `IdleExitTimeout` parameter. We don’t care about this parameter, but we are using it to trigger the cups restart, which is a privileged action.
    
5.  The file `/etc/cups/cups-files.conf` will now be world writable. Modify this file to set the `Group` parameter to `netdev`. We don’t actually need to set the `User` parameter for our chain.
    
6.  Repeat step 4 to trigger a restart of cups, causing the `Group` parameter in `cups-files.conf` to take effect.
    
7.  Stage for `wpa_supplicant` by creating a configuration file that defines the `opensc_engine_path` pointing to our shared object and creating a Python script to call the `wpa_supplicant`’s `CreateInterface` method. Because of the required data types of this method, it is easier to use Python than `dbus-send`.
    
8.  Install a new printer into cups using the `PrinterAddWithPpdFile`’s DBus method. The PPD file passed here will define the `FoomaticRIPCommandLine` parameter to execute the Python script created in step 7.
    
9.  Print a sample PDF file using the newly created printer. This will trigger the Python script, which will call the `CreateInterface` method, which will cause `wpa_supplicant` to load our shared object defined in our configuration file. This shared object will set Python to be setuid root as soon as it is loaded (using the gcc `constructor` attribute).
    
10.  Using the now-changed setuid Python, clean up the changes we have made. Remove any new lines from the various cups configuration files, reset the permissions of these files to their originals, and restart cups so everything takes effect.
    
11.  Finally, provide an interactive root shell to the user to finish the proof of concept.
    

Or, as a diagram:

![](https://snyk.io/_next/image/?url=https%3A%2F%2Fres.cloudinary.com%2Fsnyk%2Fimage%2Fupload%2Fv1725899741%2Fblog-ubuntu-chart.png&w=2560&q=75)

Conclusion
----------

In this post, we have seen that it only takes the leveraging of one small vulnerability, combined with a number of features, to achieve a chain of exploitation resulting in a full privilege escalation. Even where security controls are in place preventing the direct exploitation of a small vulnerability it may still be possible to finesse limited exploitation potential into a much greater impact.

There are a few key takeaways that I find very useful to bear in mind when performing this kind of research. The first is trying to be as holistic as possible. When looking at a system as complex as an entire Linux distribution, it pays to consider the system as a whole rather than focusing on one component. If I had limited my scope to cups alone, even with the multiple items I identified, I may not have been successful in escalating all the way to root. Still valuable research, to be sure, but not as good as it turned out to be. By keeping my eyes open and not showing favoritism to a single area, it’s possible to pivot to a completely separate area of the system (from printing to networking) to get the results I’d hoped.

The second key takeaway is that it is not necessary to find everything in order. Especially when it comes to features, it’s very valuable to have notes or prior knowledge of “if I’m in situation X, I could use feature Y to achieve result Z”. Some of the features used in this chain have been around for years, ready to be dropped into your PoC as the next step. Even when looking at something new to you, don’t make assumptions about what is achievable or realistic. You never know when you’ll end up in a weird jail as a low-privilege user, but if you see something in that context, keep it in mind when you’re gluing things together. Do not be afraid to temporarily decrease your privileges if it might help you get higher privileges on the way out. We (arguably) did this here by dropping privileges from rory:sudo to lp:netdev, because we knew lp:netdev could end up higher than rory:sudo could on its own.

We want to end with a shout-out to both the Ubuntu Security team and the cups team, who responded quickly to the reports. A special shout out to the cups team who were able to produce a patch on the same day as the report which both fixes the direct vulnerability but also added additional security measures, which would have made it doubly difficult to exploit in the first place.

Timeline
--------

08-May-2024: Reported the vulnerability chain to Ubuntu Security

23-May-2024: wpa_supplicant issue assigned [CVE-2024-5290](https://security.snyk.io/vuln/SNYK-UNMANAGED-WPASUPPLICANT-7644387) by Ubuntu security

24-May-2024: [Reported](https://github.com/OpenPrinting/cups/security/advisories/GHSA-vvwp-mv6j-hw6f) the cups specific vulnerability to OpenPrinting/cups at Ubuntu’s request. GHSA-vvwp-mv6j-hw6f

24-May-2024: cups proposed a suitable patch for the reported vulnerability

24-May-2024: cups issue assigned [CVE-2024-35235](https://security.snyk.io/vuln/SNYK-UNMANAGED-OPENPRINTINGCUPS-7254925) by Github

30-May-2024: wpa_supplicant [bug](https://bugs.launchpad.net/ubuntu/+source/wpa/+bug/2067613) opened by Ubuntu Security

03-June-2024: cups announced upcoming advisory release to 

11-June-2024: cups released patch and advisory

26-July-2024: wpa_supplicant bug patch created

06-Aug-2024: wpa_supplicant patch released

Mitigations
-----------

Cups mitigated the identified vulnerability by removing the call to chmod entirely and relying on umask to ensure the permissions are correct. Additionally, when cupsd is configured to be started on-demand (i.e. via systemd socket activation), cupsd will not process the Listen directives itself.

The wpa_supplicant project mitigated this issue by implementing a mandatory path prefix to ensure that all loaded shared objects are under the /usr/lib directory. This ensures that attacker-controlled code cannot be loaded even with sufficient access to the DBus interface.