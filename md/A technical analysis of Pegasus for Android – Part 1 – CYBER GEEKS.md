> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [cybergeeks.tech](https://cybergeeks.tech/a-technical-analysis-of-pegasus-for-android-part-1/)

> Summary

Summary

Pegasus is a spyware developed by the NSO group that was repeatedly analyzed by [Amnesty International](https://www.amnesty.org/en/latest/research/2021/07/forensic-methodology-report-how-to-catch-nso-groups-pegasus/) and [CitizenLab](https://citizenlab.ca/2020/12/the-great-ipwn-journalists-hacked-with-suspected-nso-group-imessage-zero-click-exploit/). In this article, we dissect the Android version that was initially analyzed by Lookout in this [paper](https://info.lookout.com/rs/051-ESQ-475/images/lookout-pegasus-android-technical-analysis.pdf), and we recommend reading it along with this post. During our research about Pegasus for Android, we’ve found out that vendors wrongly attributed some undocumented APK files to Pegasus, as highlighted by a researcher [here](https://twitter.com/maldr0id/status/1492520053702602755). We’ve splitted the analysis into 3 parts because of the code’s complexity and length. We’ve also tried to keep the sections name proposed by Lookout whenever it was possible so that anybody could follow the two approaches more easily. In this part, we’re presenting the initialization of the application (including its configuration), the targeted applications, the commands related to the core functionality, and the methods that Pegasus could use to remove itself from a device. Our contributions consist of dissecting the application deeper than before and explaining additional functionalities that were identified.

![](https://cybergeeks.tech/wp-content/uploads/2022/08/Pegasus-1024x585.png)

Technical analysis

SHA256: ade8bef0ac29fa363fc9afd958af0074478aef650adeb0318517b48bd996d5d5

We’ve performed the analysis using JD-GUI Java Decompiler and Android Studio.

**Initial Launch and Configuration**

The application must obtain an initial configuration from an URL found in the Browser history or from a file called “/data/myappinfo” or “/system/ttg”. As we’ll see during the entire analysis, the malware is pretty noisy and logs messages using the Log.i method. Interestingly, the author mentions the JigglyPuff character from Pokemon in the logging function:

![](https://cybergeeks.tech/wp-content/uploads/2022/08/1.jpg)Figure 1 ![](https://cybergeeks.tech/wp-content/uploads/2022/08/2.jpg)Figure 2

Firstly, the application tries to parse a config file called “/data/myappinfo” or “/system/ttg” if the first one doesn’t exist:

![](https://cybergeeks.tech/wp-content/uploads/2022/08/3.jpg)Figure 3 ![](https://cybergeeks.tech/wp-content/uploads/2022/08/4-1024x873.jpg)Figure 4

The malware deletes a file called “/data/cksnb.dat” using a binary called “/system/csk”. According to Lookout, this file is present on devices that were previously rooted. The purpose of this binary is to run a command passed as a parameter (such as “rm”) with root privileges:

![](https://cybergeeks.tech/wp-content/uploads/2022/08/5.jpg)Figure 5

Whether the process finds one of the configuration files mentioned above, it reads an URL that will be deleted from the Browser history. Some of the settings are Base64-encoded and will be decoded using an implementation of the Base64 algorithm:

![](https://cybergeeks.tech/wp-content/uploads/2022/08/6.jpg)Figure 6

The application reads a token and an “installation” value from the configuration file:

![](https://cybergeeks.tech/wp-content/uploads/2022/08/7.jpg)Figure 7

Furthermore, the process reads a configuration value called “local” and another one called “userNetwork”, which represents the mobile country code of the victim’s phone:

![](https://cybergeeks.tech/wp-content/uploads/2022/08/8.jpg)Figure 8

The malware reads a configuration value that specifies whether it should communicate with the C2 server while the phone is roaming. The configuration file also contains commands to be executed by Pegasus:

![](https://cybergeeks.tech/wp-content/uploads/2022/08/9.jpg)Figure 9

The version of the installed agent (“packageVersion”) and a value called “vulnarbilityIndicator” are also read from the configuration file. The last value is set when requesting an update package, and it would create an mp3 file that exploits the Media Player on the phone, according to our analysis:

![](https://cybergeeks.tech/wp-content/uploads/2022/08/10.jpg)Figure 10

The agent copies a file called “/data/cksnb.dat” to “/data/data/com.network.android/output.mp3”. As also highlighted below, the mp3 file would exploit a “vulnarbility” [sic] in Media Player:

![](https://cybergeeks.tech/wp-content/uploads/2022/08/11.jpg)Figure 11 ![](https://cybergeeks.tech/wp-content/uploads/2022/08/12-1024x535.jpg)Figure 12

A config value called “url address” will be removed from the browser history. The application extracts a table containing both bookmarks and history items from Browser.BOOKMARKS_URI:

![](https://cybergeeks.tech/wp-content/uploads/2022/08/13-1024x961.jpg)Figure 13

As we mentioned before, the application could retrieve the configuration from an URL containing “rU8IPXbn” found in the Browser history, as highlighted in figure 14.

![](https://cybergeeks.tech/wp-content/uploads/2022/08/14.jpg)Figure 14

The agent parses the target URL and extracts a token (“t=” parameter), a Base64-encoded command and signature (“&c”), the “installation” value (“&b”), the “userNetwork” configuration option (“&d”), and the “windowYuliyus” value (“&r”):

![](https://cybergeeks.tech/wp-content/uploads/2022/08/15.jpg)Figure 15

The malware expects a valid token and a valid target URL; otherwise, it calls the Log.e method with the “getSettingsFromBH no valid settings on getSettingsFromHistory” message:

![](https://cybergeeks.tech/wp-content/uploads/2022/08/16.jpg)Figure 16

The agent calls a function that validates the MCC (mobile country code) extracted from the configuration:

![](https://cybergeeks.tech/wp-content/uploads/2022/08/17.jpg)Figure 17

The getSubscriberId function is utilized to extract the unique subscriber ID. The first 3 digits represent the mobile country code (MCC), which is compared with the value extracted from the configuration:

![](https://cybergeeks.tech/wp-content/uploads/2022/08/18.jpg)Figure 18

The application verifies whether the “did_we_restart_after_upgrade_already” value, which indicates that the device was rebooted after the installation of Pegasus, is true or false. In the case of no reboot, the application restarts the phone using the “reboot” command:

![](https://cybergeeks.tech/wp-content/uploads/2022/08/19.jpg)Figure 19

Figure 20 reveals how the “/system/csk” binary is used to run commands with root privileges:

![](https://cybergeeks.tech/wp-content/uploads/2022/08/20.jpg)Figure 20

The agent uses a regex to parse the target URL and to extract the IP address and the token, as highlighted below:

![](https://cybergeeks.tech/wp-content/uploads/2022/08/21.jpg)Figure 21 ![](https://cybergeeks.tech/wp-content/uploads/2022/08/22.jpg)Figure 22

The process calls a function that deletes the target URL from the Browser history based on the IP address extracted above:

![](https://cybergeeks.tech/wp-content/uploads/2022/08/23.jpg)Figure 23

The malware starts logging some messages before performing the deletion operation:

![](https://cybergeeks.tech/wp-content/uploads/2022/08/24.jpg)Figure 24

The application retrieves the bookmarks and history items from “Browser.BOOKMARKS_URI” and “Browser.HISTORY_PROJECTION”:

![](https://cybergeeks.tech/wp-content/uploads/2022/08/25-1024x793.jpg)Figure 25

The URL containing “rU8IPXbn” is deleted from the Browser history by calling the Browser.deleteFromHistory function:

![](https://cybergeeks.tech/wp-content/uploads/2022/08/26.jpg)Figure 26 ![](https://cybergeeks.tech/wp-content/uploads/2022/08/27.jpg)Figure 27

The configuration data is saved in a preference file called “NetworkPreferences” that can be accessed using the Android SharedPreferences APIs.

**Targeted Applications**

1. Facebook

The database files storing the Facebook messages are made accessible to everyone by running the following commands:

*   chmod 0777 /data/data/com.facebook.katana
*   chmod 0777 /data/data/com.facebook.katana/databases
*   chmod 0777 /data/data/com.facebook.katana/databases/threads_db2
*   chmod 0777 /data/data/com.facebook.katana/databases/threads_db2-journal

The following SQL query is executed:

*   SELECT messages.msg_id, messages.thread_id, messages.timestamp_ms, messages.text, messages.sender, threads.participants from messages INNER JOIN threads ON messages.thread_id=threads.thread_id

![](https://cybergeeks.tech/wp-content/uploads/2022/08/28-1024x330.jpg)Figure 28

2. Kakao

The database files storing the KakaoTalk chat logs are made accessible to everyone by running the following commands:

*   chmod 0777 /data/data/com.kakao.talk
*   chmod 0777 /data/data/com.kakao.talk/databases
*   chmod 0777 /data/data/com.kakao.talk/databases/KakaoTalk.db
*   chmod 0777 /data/data/com.kakao.talk/databases/KakaoTalk.db-journal

The following SQL query is executed:

*   SELECT chat_logs.id, chat_logs.chat_id, chat_logs.created_at, chat_logs.message, chat_logs.user_id, chat_logs.type, c.members FROM chat_logs JOIN chat_rooms c ON chat_logs.chat_id=c.id

![](https://cybergeeks.tech/wp-content/uploads/2022/08/29-1024x202.jpg)Figure 29

3. Skype

The database files storing the Skype chat messages are made accessible to everyone by running the following commands:

*   chmod 0777 /data/data/com.skype.raider/files/<Directories>
*   chmod 0777 /data/data/com.skype.raider/files/main.db
*   chmod 0777 /data/data/com.skype.raider/files/main.db-journal

The following SQL query is executed:

*   SELECT Messages.id as msg_id, messages.convo_id, from_dispname, messages.author, messages.timestamp, messages.body_xml, conversations.displayname, Messages.dialog_partner FROM Messages LEFT JOIN Conversations ON messages.convo_id = conversations.id

![](https://cybergeeks.tech/wp-content/uploads/2022/08/30-1024x590.jpg)Figure 30

4. Twitter

The database files storing the Twitter messages are made accessible to everyone by running the following commands:

*   chmod 0777 /data/data/com.twitter.android
*   chmod 0777 /data/data/com.twitter.android/databases/*.db

The following SQL query is executed:

*   SELECT messages._id, messages.type, messages.msg_id, messages.content, messages.created, messages.sender_id, messages.recipient_id, messages.thread, s.name, s.username, r.name, r.username FROM messages JOIN users s ON messages.sender_id = s.user_id JOIN users r ON messages.recipient_id = r.user_id

![](https://cybergeeks.tech/wp-content/uploads/2022/08/31-1024x377.jpg)Figure 31

5. Viber

The database files storing the Viber messages are made accessible to everyone by running the following commands:

*   chmod 0777 /data/data/com.viber.voip/
*   chmod 0777 /data/data/com.viber.voip/databases/viber_messages
*   chmod 0777 /data/data/com.viber.voip/databases/viber_messages-journal

The agent executes the SQL query displayed in the figure below.

![](https://cybergeeks.tech/wp-content/uploads/2022/08/32-1024x390.jpg)Figure 32

6. WhatsApp

The database files storing the WhatsApp messages are made accessible to everyone by running the following commands:

*   chmod 0777 /data/data/com.whatsapp
*   chmod 0777 /data/data/com.whatsapp/databases
*   chmod 0777 /data/data/com.whatsapp/shared_prefs
*   chmod 0777 /data/data/com.whatsapp/shared_prefs/com.whatsapp_preferences.xml
*   chmod 0777 /data/data/com.whatsapp/databases/msgstore.db
*   chmod 0777 /data/data/com.whatsapp/databases/wa.db

The following SQL queries are executed:

*   select * from messages
*   select timestamp from messages order by _id desc limit 1

![](https://cybergeeks.tech/wp-content/uploads/2022/08/33.jpg)Figure 33 ![](https://cybergeeks.tech/wp-content/uploads/2022/08/34-1024x590.jpg)Figure 34

7. Gmail

The following SQL queries are executed (“/data/data/com.google.android.gm/databases/EmailProvider.db” database):

*   select * from messages
*   select * from Message
*   select _id from messages order by _id desc limit 1
*   select _id from Message order by _id desc limit 1

![](https://cybergeeks.tech/wp-content/uploads/2022/08/35.jpg)Figure 35 ![](https://cybergeeks.tech/wp-content/uploads/2022/08/36.jpg)Figure 36

8. Android Native Email

The database file storing the Android Native emails and attachments is made accessible to everyone by running the following commands:

*   chmod 0777 /data/data/com.android.email
*   chmod 0777 /data/data/com.android.email/databases/EmailProvider.db

The following SQL queries are executed:

*   select * from Message where _id = <Id>
*   select * from Attachment where _id = <Id>

![](https://cybergeeks.tech/wp-content/uploads/2022/08/37-1.jpg)Figure 37 ![](https://cybergeeks.tech/wp-content/uploads/2022/08/38-1-1024x669.jpg)Figure 38

9. Android Native Browser

The database file storing the user name and password entered in the WebView is made accessible to everyone by running the following commands:

*   chmod 0777 /data/data/com.android.browser
*   chmod 0777 /data/data/com.android.browser/databases
*   chmod 0777 /data/data/com.android.browser/databases/webview.db

The following SQL query is executed:

*   select * from password

The agent also extracts the bookmarks and history items from Browser.BOOKMARKS_URI and Browser.HISTORY_PROJECTION.

![](https://cybergeeks.tech/wp-content/uploads/2022/08/39.jpg)Figure 39 ![](https://cybergeeks.tech/wp-content/uploads/2022/08/40-1024x600.jpg)Figure 40 ![](https://cybergeeks.tech/wp-content/uploads/2022/08/41-1024x650.jpg)Figure 41

10. Default Calendar

The agent extracts the Android version string from the “Build.VERSION.RELEASE” property:

![](https://cybergeeks.tech/wp-content/uploads/2022/08/42.jpg)Figure 42

Depending on the Android version, the calendar’s events can be found at “content://com.android.calendar/events” or “content://calendar/events”, as shown in figure 43:

![](https://cybergeeks.tech/wp-content/uploads/2022/08/43.jpg)Figure 43

The application extracts the events title, summary, description, and other properties (see figure 44). The calendar entries are added to an XmlSerializer object that has multiple attributes:

![](https://cybergeeks.tech/wp-content/uploads/2022/08/44-1024x174.jpg)Figure 44

**Suicide Functionality**

1st method

The agent will kill itself if it finds a file called “/sdcard/MemosForNotes” on the device:

![](https://cybergeeks.tech/wp-content/uploads/2022/08/45.jpg)Figure 45

The malware starts the removal operation by logging the “removeAppalication start” message, as shown in the figure below:

![](https://cybergeeks.tech/wp-content/uploads/2022/08/46.jpg)Figure 46

The application removes all files that are located in the “/data/local/tmp/ktmu” directory:

![](https://cybergeeks.tech/wp-content/uploads/2022/08/47.jpg)Figure 47

The SharedPreferences.Editor.clear function is utilized to remove the following preference files containing configuration data: “NetworkPreferences”, “NetworkWindowAddresess”, and “NetworkDataList” (see figure 48).

![](https://cybergeeks.tech/wp-content/uploads/2022/08/48.jpg)Figure 48

The following commands are run by the malware:

*   export LD_LIBRARY_PATH=/vendor/lib:/system/lib – set the LD_LIBRARY_PATH environment variable

*   mount -o remount,rw /dev/null /system – remount the system directory

*   force-stop com.network.android – stop the malicious application

*   disable com.network.android – disable the malicious application

*   rm /system/app/com.media.sync.apk – remove this file

*   pm uninstall com.network.android – uninstall the malicious application

*   rm /system/ttg – remove the configuration file

*   chmod 0777 /system/csk; rm /system/csk – delete the binary used to run commands with root privileges (“superuser binary”)

![](https://cybergeeks.tech/wp-content/uploads/2022/08/49.jpg)Figure 49

2nd method

The malware will also remove itself if it didn’t communicate with the C2 server in the last 60 days:

![](https://cybergeeks.tech/wp-content/uploads/2022/08/50.jpg)Figure 50

3rd method

The suicide functionality is also used when no subscriber ID is identified (see figure 18).

4th method

The Pegasus’ C2 server can issue a command to the agent in order to remove itself.

Pegasus implements 4 main commands that can be executed: dumpCmd, upgradeCmd, camCmd, and emailAttCmd (figure 51).

![](https://cybergeeks.tech/wp-content/uploads/2022/08/51.jpg)Figure 51

**dumpCmd command**

The agent can activate/deactivate some actions depending on the dumpSms, dumpWhatsApp, dumpEmails, dumpContacts, dumpCalander [sic], dumpCall boolean values:

![](https://cybergeeks.tech/wp-content/uploads/2022/08/52.jpg)Figure 52

The agent extracts the SMS messages sent and received from “content://sms/sent” and “content://sms/inbox”. Every SMS is stored in a new XmlSerializer object, as highlighted below:

![](https://cybergeeks.tech/wp-content/uploads/2022/08/53.jpg)Figure 53 ![](https://cybergeeks.tech/wp-content/uploads/2022/08/54.jpg)Figure 54

The malware extracts the Phone call logs from “CallLog.Calls.CONTENT_URI”. Each log is stored in a new XmlSerializer object that contains attributes such as “recordId”, “timestamp”, “type”, “number”, “duration”, “isStart”, and “direction”:

![](https://cybergeeks.tech/wp-content/uploads/2022/08/55-1024x508.jpg)Figure 55 ![](https://cybergeeks.tech/wp-content/uploads/2022/08/56-1024x329.jpg)Figure 56

The application retrieves the phone contacts from the “ContactsContract.Contacts.CONTENT_VCARD_URI” entry. Every contact is stored in an XmlSerializer object:

![](https://cybergeeks.tech/wp-content/uploads/2022/08/57-1024x130.jpg)Figure 57 ![](https://cybergeeks.tech/wp-content/uploads/2022/08/58-1024x484.jpg)Figure 58 ![](https://cybergeeks.tech/wp-content/uploads/2022/08/59.jpg)Figure 59

The agent obtains a list of application processes running on the device via a call to getRunningAppProcesses:

![](https://cybergeeks.tech/wp-content/uploads/2022/08/60-1024x243.jpg)Figure 60

This command is also responsible for retrieving the information already described in the “Targeted applications” section.

**upgradeCmd command**

The process creates a file called “/data/data/com.network.android/upgrade/uglmt.dat” that will store the upgraded application that is downloaded from the C2 server. It computes the MD5 hash of the file and then compares the result with a value that comes with the package:

![](https://cybergeeks.tech/wp-content/uploads/2022/08/61.jpg)Figure 61 ![](https://cybergeeks.tech/wp-content/uploads/2022/08/62.jpg)Figure 62

The agent loads uglmt.dat as a dex file using DexClassLoader and then calls the “com.media.provapp.DrivenObjClass.perfU” method with the “/data/data/com.network.android/upgrade/intro.mp3” argument:

![](https://cybergeeks.tech/wp-content/uploads/2022/08/63-1024x68.jpg)Figure 63 ![](https://cybergeeks.tech/wp-content/uploads/2022/08/64-1024x798.jpg)Figure 64

After successfully upgrading the agent, the following files are deleted by calling the File.delete function:

*   /data/data/com.network.android/upgrade/uglmt.dat

*   /data/data/com.network.android/upgrade/cuvmnr.dat

*   /data/data/com.network.android/upgrade/zero.mp3

*   /data/data/com.network.android/upgrade/*com.media.sync*

![](https://cybergeeks.tech/wp-content/uploads/2022/08/65.jpg)Figure 65

**camCmd command**

Firstly, the process relies on a binary that should exist on Android called “/system/bin/screencap” in order to take a screenshot of the screen. The result is saved as a PNG image at “/data/data/com.network.android/bqul4.dat”:

![](https://cybergeeks.tech/wp-content/uploads/2022/08/66.jpg)Figure 66

Secondly, if the above binary doesn’t exist then the malware will use a resource (found in res/raw/ folder) called “take_screen_shot” that will handle the operation. The result is stored at “/data/data/com.network.android/tss64.dat”:

![](https://cybergeeks.tech/wp-content/uploads/2022/08/67.jpg)Figure 67

In any case, the screenshots’ information will be stored in an XmlSerializer object described below:

![](https://cybergeeks.tech/wp-content/uploads/2022/08/68-1024x272.jpg)Figure 68

As we can see in the assembly code of take_screen_shot, the binary captures the Android screen content by reading the “/dev/graphics/fb0” framebuffer that is processed and stored as a PNG image:

![](https://cybergeeks.tech/wp-content/uploads/2022/08/69.jpg)Figure 69 ![](https://cybergeeks.tech/wp-content/uploads/2022/08/70.jpg)Figure 70

The captured PNG images are compressed to JPG and saved as “ScreenShot-res<Integer>-<Current Time in seconds>.jpg”:

![](https://cybergeeks.tech/wp-content/uploads/2022/08/71-1024x894.jpg)Figure 71

The process can take photos with the front/back camera. It calls the getParameters function in order to obtain the current settings for the Camera service and then calls getPreviewFormat:

![](https://cybergeeks.tech/wp-content/uploads/2022/08/72.jpg)Figure 72

Depending on the front/back camera, the taken photo is saved as “Front-res<Integer>-<Current Time in seconds>.jpg” or “Back-res<Integer>-<Current Time in seconds>.jpg”

![](https://cybergeeks.tech/wp-content/uploads/2022/08/73.jpg)Figure 73

**emailAttCmd command**

The application creates a database containing a table called “NetworkData” that stores the email attachments’ name and path, as displayed in figure 74.

![](https://cybergeeks.tech/wp-content/uploads/2022/08/74-1024x223.jpg)Figure 74

The command’s purpose is to extract a specific email attachment mentioned by the C2 server:

![](https://cybergeeks.tech/wp-content/uploads/2022/08/75-1024x778.jpg)Figure 75 ![](https://cybergeeks.tech/wp-content/uploads/2022/08/76.jpg)Figure 76

We were surprised to find out that during the initialization routine, the threat actor mentioned the “Pegasus” string in a log message:

![](https://cybergeeks.tech/wp-content/uploads/2022/08/77.jpg)Figure 77