> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [medium.com](https://medium.com/mobis3c/insecure-data-storage-insecure-logging-storage-bf50db1c4879)

> Before we get started, make sure you have genymotion setup ready. if not follow this guide to setup &......

Insecure Data Storage: Insecure Logging & Storage
=================================================

[![](https://miro.medium.com/fit/c/96/96/1*uTzz4c17GKaaSp47mMSkgw.jpeg)](https://3kal.medium.com/?source=post_page-----bf50db1c4879--------------------------------)[Kal](https://3kal.medium.com/?source=post_page-----bf50db1c4879--------------------------------)Follow[Feb 12](/mobis3c/insecure-data-storage-insecure-logging-storage-bf50db1c4879?source=post_page-----bf50db1c4879--------------------------------) · 5 min read![](https://miro.medium.com/max/1400/0*35mJaLH5jFWd-JHx.jpg)Insecure logging

Before we get started, make sure you have genymotion setup ready. if not follow this guide to setup & configure the [genymotion](/mobis3c/setting-up-an-android-pentesting-environment-29991aa0c3f1#cc07) in Linux.

PID Cat
=======

**Process ID (PID) Cat** is a logcat script which only shows log entries for processes from a specific application package.

Installing PID Cat:

```
#Download pidcat & make it executable
sudo wget -O /usr/local/bin/pidcat https://raw.githubusercontent.com/JakeWharton/pidcat/master/pidcat.py && sudo chmod +x /usr/local/bin/pidcat

```

To use PIDCat, you need to pass app identifier which is unique for each application. ex: for whatsapp, com.whatsapp

![](https://miro.medium.com/max/60/1*IiaVUI_Yf93bZ0eSRO7nSQ.png?q=20)![](https://miro.medium.com/max/875/1*IiaVUI_Yf93bZ0eSRO7nSQ.png)![](https://miro.medium.com/max/1400/1*IiaVUI_Yf93bZ0eSRO7nSQ.png)app identifier

Now, let us learn how to use this PIDCat for identifying insecure logging.

Launch your preferred device in Genymotion and make sure it is accessible through adb and make sure device is connected to network and ip is assigned.

![](https://miro.medium.com/max/60/1*V290lAW4EBZfBnnv6vNpWw.png?q=20)![](https://miro.medium.com/max/875/1*V290lAW4EBZfBnnv6vNpWw.png)![](https://miro.medium.com/max/1400/1*V290lAW4EBZfBnnv6vNpWw.png)adb devices

Before using PIDCat, let us understand the logs using logcat. for that we need to connect to the device by using adb serial connection.

![](https://miro.medium.com/max/60/1*Mcn-amE7npGeJseWNwnwVw.png?q=20)![](https://miro.medium.com/max/875/1*Mcn-amE7npGeJseWNwnwVw.png)![](https://miro.medium.com/max/1400/1*Mcn-amE7npGeJseWNwnwVw.png)adb logcat![](https://miro.medium.com/max/60/1*344vtamrPlUAUdu_PE472w.png?q=20)![](https://miro.medium.com/max/875/1*344vtamrPlUAUdu_PE472w.png)![](https://miro.medium.com/max/1400/1*344vtamrPlUAUdu_PE472w.png)log

Format of the above log message is as follows:

1.  The timestamp of the log messages
2.  The process id of where log messages come from
3.  The thread id
4.  Type of Log

```
Priority value is one of the following character values, ordered from lowest to highest priority:

V — Verbose (lowest priority)*
D — Debug*
I — Info*
W — Warning*
E — Error*
A — Assert(highest priority)

```

5. Service that is pushing the log

6. log details

Now, let us see the same logs using PIDCat to keep it clean which shows only service, type and log.

```
pidcat -s 192.168.56.101:5555

```

![](https://miro.medium.com/max/60/1*r2vtnSBURJANkyLzAT7_-w.png?q=20)![](https://miro.medium.com/max/875/1*r2vtnSBURJANkyLzAT7_-w.png)![](https://miro.medium.com/max/1400/1*r2vtnSBURJANkyLzAT7_-w.png)PIDCat log

As we saw, how pidcat simplifies the logs. Let us use pidcat to capture the logs of an application to find sensitive information which are being logged.

we can capture logs of specific application using pidcat by giving the application package name.

```
pidcat -s 192.168.56.101:5555 com.google.android

```

![](https://miro.medium.com/max/60/1*7a0VrrW_T3WfAQtP-XkJZA.png?q=20)![](https://miro.medium.com/max/875/1*7a0VrrW_T3WfAQtP-XkJZA.png)![](https://miro.medium.com/max/1400/1*7a0VrrW_T3WfAQtP-XkJZA.png)

The above image consists of the log which has been logged by an application, which shows a bearer token to authenticate the user made to the api by http library.

**What can happen here?**

If any other application present on the device has “read log” access then they can use this token to act as the legit user.

But Android doesn’t support this permission for use by third-party applications anymore, because Log entries can contain the user’s private information.

Still applications, which are pre-installed/installed on privileged partition or have root access can read this logs.

**Can we report this?**

**No**, “Read_Logs” permission is revoked for third-party applications after Android 4.1 and Most applications doesn’t support Android 4.1 or less.

But while dealing with logs, remember to crash the application by closing the emulator or just clear the application by swiping, while capturing the logs.

Sometimes when the application gets crashed mid-way, the application logs a crash dump into a public readable directories ex: Downloads folder

As any application can read from this kind of directories and an application when crashed or willingly creating log files here allows other applications to read the logs which may contain sensitive information logged during the crash.

**Finding Files with global read permissions**

We will be using rooted android device/emulator for Security testing as it provides access for entire Android system.

![](https://miro.medium.com/max/60/1*zoNeVBfZ1LU7672st8sl9g.png?q=20)![](https://miro.medium.com/max/796/1*zoNeVBfZ1LU7672st8sl9g.png)![](https://miro.medium.com/max/1274/1*zoNeVBfZ1LU7672st8sl9g.png)rooted shell

Now, Let us list the files present in `/data/data` using ls -al

![](https://miro.medium.com/max/60/1*fDVZP5SV_z1tWqvcJPRZBg.png?q=20)![](https://miro.medium.com/max/875/1*fDVZP5SV_z1tWqvcJPRZBg.png)![](https://miro.medium.com/max/1400/1*fDVZP5SV_z1tWqvcJPRZBg.png)privileged partition

Using rooted device allows us to access the files of the apps which are not accessible with non-rooted devices.

we can find public readable files in the privileged directory which has read access for third-party apps by using find

```
find . -perm -o+r

```

![](https://miro.medium.com/max/60/1*C9vxMA1bhO20rSEmvLuKfQ.png?q=20)![](https://miro.medium.com/max/875/1*C9vxMA1bhO20rSEmvLuKfQ.png)![](https://miro.medium.com/max/1400/1*C9vxMA1bhO20rSEmvLuKfQ.png)public readable files

search for files from the result of above command containing sensitive information and **report it** (if sensitive information found)

Now, Let us see how applications can access the files and use sensitive information available in that file.

Let us impersonate as a different app and try to access the file created by other application but have public readable files.

Above we have found many files which have public readable access. so let us try to access the file by impersonating as normal application.

we will take one uid from the running process to impersonate and switch our user to the apps user id.

![](https://miro.medium.com/max/60/1*ucL8dhGgd_fGS83S8uAw2w.png?q=20)![](https://miro.medium.com/max/805/1*ucL8dhGgd_fGS83S8uAw2w.png)![](https://miro.medium.com/max/1288/1*ucL8dhGgd_fGS83S8uAw2w.png)processes![](https://miro.medium.com/max/60/1*hwsQPrB-7N8hsDNcaBO5EQ.png?q=20)![](https://miro.medium.com/max/855/1*hwsQPrB-7N8hsDNcaBO5EQ.png)![](https://miro.medium.com/max/1368/1*hwsQPrB-7N8hsDNcaBO5EQ.png)impersonate

After impersonating, when we try to access the `/data/data` directory we get permission denied but still we can access the public readable file `kal.txt` which is present in that privileged directory.

![](https://miro.medium.com/max/60/1*I760OXuq3lW_DFlmOtq8_w.png?q=20)![](https://miro.medium.com/max/531/1*I760OXuq3lW_DFlmOtq8_w.png)![](https://miro.medium.com/max/850/1*I760OXuq3lW_DFlmOtq8_w.png)permission denied![](https://miro.medium.com/max/60/1*U_FA3gvT65qYTZtBDU3QfQ.png?q=20)![](https://miro.medium.com/max/464/1*U_FA3gvT65qYTZtBDU3QfQ.png)![](https://miro.medium.com/max/742/1*U_FA3gvT65qYTZtBDU3QfQ.png)public readable file in privileged directory

**Summary:**

As a root, we can see the files created by the application which may contain the sensitive information. so as a root we find the permission of the files created by that application and if it is public readable then we impersonate as different app to show that other apps can access the files created by the application which contains sensitive information.

If application logs sensitive data but are not ready to accept as a valid bug as we accessed it as a root user we impersonate as normal user (other app) to make it valid.