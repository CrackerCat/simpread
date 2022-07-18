> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [thebinaryhick.blog](https://thebinaryhick.blog/2022/07/14/session-on-android-an-app-wrapped-in-signal/)

> NOTE: parts of this article describe steps by which the order of encryption methods are reversed to r......

_NOTE: parts of this article describe steps by which the order of encryption methods are reversed to render encrypted data in clear-text. This was done in order to investigate the app being discussed. The Signal & Session protocols have not been “defeated,” “hacked,” or “broken.” Both are still as robust as they were before this article was written._

![](https://media.giphy.com/media/zQ3Otg91WjqRKHcTUD/giphy.gif)An accurate depiction of me trying to decrypt stuff.

With the recent (or not) political instability in the United States my Twitter feed has, once again, become filled with discussions around secure messaging. Many folks talk about their messaging platform of choice and why they feel it is more secure than other available platforms. One app in particular, Session, caught my attention as it was mentioned in the same sentence as Signal, which is saying something. And it wasn’t a one-off either; many people said they preferred Session. I had not heard of this app before, so I decided to take a look at its offering on Android.

I hit a couple of roadblocks while investigating this app. I have included discussion on those road blocks in hope that I save at least one examiner from some of the headache I experienced.

God of Mischief
---------------

Session is rooted in the cryptocurrency realm. It started life in 2018 as Loki Messenger, which was part of the Loki network/cryptocurrency platform. Loki was designed to let users interact with the wider Internet in a decentralized way. It took a page from TOR by routing its traffic through multiple service nodes before reaching its intended destination (simply called “onion routing”). Loki Messenger also used onion routing, which was (and is) great for privacy, and added the Signal protocol for an extra pinch of privacy.

In 2019 Loki Messenger was rebranded to Session Private Messenger and was released in early 2020. The new app still sported the Signal protocol, used the Loki network as a backend (rebranded as Oxen), and continued to add functionality as time wore on (e.g. P2P calling).

Session is available on Android, F-Droid, iOS, Linux, macOS, and Windows. Additionally, the Android .apk file is available for download directly from the Session GitHub repo if you don’t feel like downloading from the Google Play Store.

Poking Around
-------------

In addition to borrowing the Signal protocol, Session also borrows some of its aesthetics and functionality from Signal. Figure 1 shows the landing pages for each app, Figure 2 shows their chat windows, Figure 3 shows the main portion of the Settings page, and Figure 4 shows the Privacy portion of the Settings. For each figure, Session is on the left and Signal is on the right.

![](https://thebinaryhick.files.wordpress.com/2022/07/figure-1.png?w=48)Figure 1. Landing pages.

![](https://thebinaryhick.files.wordpress.com/2022/07/figure-2.png?w=820)Figure 2. Chat windows.

![](https://thebinaryhick.files.wordpress.com/2022/07/figure-3.png?w=820)Figure 3. Settings.

![](https://thebinaryhick.files.wordpress.com/2022/07/figure-4.png?w=48)Figure 4. Privacy settings.

Figure 3 is important here, and is the main difference from a user’s privacy perspective. Signal utilizes a phone number to identify a user. In Session, a user is identified by their Session ID. Not using a phone number reduces the amount of identifying data Session holds, and likely makes the app resistant to SIM swapping (just like the Signal PIN). In order to start a chat with another party a user would need the other party’s Session ID or scan a QR code on the other party’s phone. Both can be shared via other messaging applications that may be on the device.

There are a couple of settings examiner should be aware of which are found in the Privacy portion of the Settings menu. See Figure 5.

![](https://thebinaryhick.files.wordpress.com/2022/07/figure-5.png?w=461)Figure 5. Privacy settings.

Just like Signal and other privacy-focused apps, Session will, by default, prevent screenshots from being taken and in the recent apps (setting in blue box). That means an examiner should expect to find blanks screenshots for Session in the **/USERDATA/system_ce/%USER_NUMBER%/snapshots/** directory path. Additionally, voice and video calls are off by default in the app (red box).

Session also has a self-destructing message feature. This feature is set on a per-conversation basis. Messages can have a time-to-live (TTL) anywhere between five (5) seconds and one (1) week. Figure 6 shows the Disappearing Messages setting within a conversation and how the conversation window looks after it has been set to one (1) week.

![](https://thebinaryhick.files.wordpress.com/2022/07/figure-6.png?w=820)Figure 6. TTL of one week.

Session uses onion routing to route its traffic between users using the aforementioned Oxen (formerly Loki) network. A user can see which service nodes are being used to route their traffic. See Figure 7.

![](https://thebinaryhick.files.wordpress.com/2022/07/figure-7.png?w=461)Figure 7. Service node route.

Again, this is onion routing, but it is using Oxen Service Nodes and not TOR.

The last feature of note is seen in Figure 8.

![](https://thebinaryhick.files.wordpress.com/2022/07/figure-8.png?w=820)Figure 8. Clear all data.

The Clear Data button is found in the Settings menu. Pressing presents the dialogue screen seen on the right. If a user chooses to delete their data here, it is completely wiped out. Based on my testing, all message content is deleted.

The rest of the app is very similar to other messaging apps. It can make and receive calls (audio and video) and transfer files (pictures, videos, documents, etc.). The version I tested does not have a group calling feature.

Encryption. Lots of Encryption.
-------------------------------

Session uses a healthy dose of encryption to protect message content and transferred files. Fortunately, as discussed later, there are a few resources online that aided in the investigation of this app. For testing I used a rooted Pixel 5a running Android 12 (March 2022 security patch), and a virtual Android device in [Corellium](https://www.corellium.com/?utm_term=corellium&utm_campaign=Brand:+Corellium&utm_source=adwords&utm_medium=ppc&hsa_acc=9314668052&hsa_cam=17015820365&hsa_grp=137300490193&hsa_ad=600901037258&hsa_src=g&hsa_tgt=kwd-1460996826397&hsa_kw=corellium&hsa_mt=e&hsa_net=adwords&hsa_ver=3&gclid=CjwKCAjw_b6WBhAQEiwAp4HyIJIEVzEYRsI8Y-LlZeRohQSkwDc--kSHW6faayzKBl3yrqBpuNl7_BoCe0EQAvD_BwE). The need for virtualization will become evident later in this post. The version of Sesson tested was 1.13.6 (2875).

The .apk name for Session is a carry over from its time as Loki Messenger: _network.loki.messenger_. The root of its folder contained a surprise. See Figure 9.

![](https://thebinaryhick.files.wordpress.com/2022/07/figure-9.png?w=820)Figure 9. CSVs. In the root.

This was an odd find. Naturally, I wanted to see what they contained. See Figures 10 & 11.

![](https://thebinaryhick.files.wordpress.com/2022/07/figure-10.png?w=876)Figure 10. geolite2_country_blocks_ipv4.csv.

![](https://thebinaryhick.files.wordpress.com/2022/07/figure-11.png?w=820)Figure 11. geolite2_country_locations_english.csv.

While I am not completely certain, I suspect these files aid in the onion routing feature. I did not find any evidentiary value in them and moved on.

Message content is contained in a database contained in the **~/databases/** folder. See Figure 12.

![](https://thebinaryhick.files.wordpress.com/2022/07/figure-12.png?w=820)Figure 12. Well, hello there.

This was a surprising find. Remember earlier when it was mentioned that Session is based on the Signal protocol. As it turns out, that plays a role in the forensics for the app. Examiners that have done mobile device forensics and have encountered Signal know the database that stores Signal message content in Android is also called _signal.db_. I pulled the database and had a look. See Figure 13.

![](https://thebinaryhick.files.wordpress.com/2022/07/figure-13.png?w=820)Figure 13. (Un)Lucky 13.

Just like Signal, the database for Session is encrypted. Session’s and Signal’s source code is available on GitHub, so I started there. Session’s code for Android can be found [here](https://github.com/oxen-io/session-android) and Signal can be found [here](https://github.com/signalapp/Signal-Android). Part of the Session repo contains a portion of forked Signal code, including the crypto portion.

Hi, I’d Like to Buy a Key From Your Store
-----------------------------------------

Browsing the Session repo, I ran across _KeyStoreHelper.java_ in the forked Signal code. Figure 14 shows the portion that is of interest.

![](https://thebinaryhick.files.wordpress.com/2022/07/figure-14.png?w=820)Figur4. Wait, what?

The Android Keystore is responsible for managing keys within Android. Keys are used for a variety of reasons, but in Session’s (and Signal’s) case the key is used to protect certain data, including the passphrase (the key…yes a second one) to encrypt/decrypt the database containing message content, _signal.db_. Additionally, developers can assign alias names to any keys they store in the keystore. Here, we can see in the red box Sesson’s app developers opted to assign their key the alias “SignalSecret” (String KEY_ALIAS in red box), which is the same alias of Signal’s key.

The second file I found in the Session repo that was of interest was _SQLCipherOpenHelper.java_. Figure 15 shows the relevant part of the file.

![](https://thebinaryhick.files.wordpress.com/2022/07/figure-15.png?w=820)Figure 15. Bingo.

The blue box in Figure 15 identifies the database name, _signal.db_, and identifies the the cipher parameters via the PRAGMAs (red boxes). These settings are not SQL Cipher 3 or 4, so now we know Session uses the same custom cipher as Signal.

So far, it appears Session uses the same crypto as Signal.

Prior to Android 12, keys were stored in individual files within the **USERDATA/misc/keystore/%USER_NUMBER%/** directory path, and had a specific naming format: UID_KEYTYPE_ALIAS. So, for example, one of Signal’s keys on a particular device could be named _1250_USRPKEY_SignalSecret_ with 1250 being the UID, USRPKEY being the key type, and SignalSecret being the key alias. In Android 12, however, keys are stored as blobs in the database _persistent.sqlite_, which is found in the **USERDATA/misc/keystore/** directory path. Figure 16 shows the content of _persistent.sqlite_ from the Pixel 5a.

![](https://thebinaryhick.files.wordpress.com/2022/07/figure-16.png?w=820)Figure 16. persistent.sqlite (table keyentry).

The table **keyentry** is the first place to start. Filtering on the column _alias_ using “SignalSecret” shows what is seen in Figure 17. This is the entry for Session in the Keystore.

![](https://thebinaryhick.files.wordpress.com/2022/07/figure-17.png?w=820)Figure 17. Filtered.

The value seen in the _id_ column relates to the value in the column _keyentryid_ in the **blobentry** table. Filtering by the value in **blobentry** finds a single entry. As the column name suggests, there is an associated blob value. See Figure 18.

![](https://thebinaryhick.files.wordpress.com/2022/07/figure-18.png?w=862)Figure 18. The lone entry.

Figure 19 shows the blob contents. The blob _should_ contain the key needed to decrypt _signal.db_.

![](https://thebinaryhick.files.wordpress.com/2022/07/figure-19.png?w=821)Figure 19. The blob.

Unfortunately, I was forgetting an important detail. Without going too in-depth here, the contents seen in Figure 19 contain the _encrypted_ key of Session. The Pixel 5a, like many modern Android phones, uses a hardware-backed keystore; it is similar in concept to the Secure Enclave in iOS. The Pixel series of phones, starting with the Pixel 3, use the Titan M co-processor to back the keystore.

Keys stored in the hardware are used to encrypt keys used by apps and, once encrypted, those app keys are stored on disk in an encrypted state (e.g. as a blob in _persistent.sqlite_). So, it’s a bit like the movie Inception here: the key needed to decrypt the encrypted Session database is, itself, encrypted. ![](https://s0.wp.com/wp-content/mu-plugins/wpcom-smileys/twemoji/2/svg/1f642.svg) If you are interested in reading more about this subject I suggest reading up on it in [Google’s developer documentation](https://source.android.com/security/keystore). Basically, I was stuck at this point. Even with root level access to the phone I was only getting the encrypted key.

Enter [Corellium](https://www.corellium.com/).

For those that are not familiar, Corellium provides virtualization services for mobile device security researchers. Their claim to fame is their ability to virtualize most modern iOS devices on modern versions of iOS. Researchers can virtualize a particular model iPhone with a compatible version of iOS and have that virtualized iPhone run in a jailbroken state. It does cost money but it has come through clutch when I’ve needed it, so, for me, it is worth the money.

They also have the ability to virtualize Android devices in a rooted state. I chose a Pixel 3 running Android 12, but, unlike my physical Pixel 3, the virtual one did not have a co-processor that could be used for a hardware-backed keystore.

Following the same steps as above, the blob for Session was extracted from _persistent.sqlite_ from the virtualized Pixel 3. Figure 20 shows the blob.

![](https://thebinaryhick.files.wordpress.com/2022/07/figure-20.png?w=812)Figure 20. Unencrypted key blob.

Again, because the virtualized device does not have a hardware-backed keystore, this blob is not encrypted. Note the additional data in the blob header starting at offset 0x08 in Figure 19. I suspect “DKMK” is indicative of a hardware-backed key, but would need to do further testing to confirm.

Fortunately, there has been a bit of academic work done on Signal. The best resource I found was [here](https://www.sciencedirect.com/science/article/pii/S2666281722000166?ref=pdf_download&fr=RR-2&rr=728c9a43dfb2b033). This paper was the best explanation of basic Signal crypto operations. Based on information in the paper and what was in the Git repo, I was able to isolate the key for Session, which was a 16-byte string starting at offset 0x0D in Figure 21 (highlighted in red).

![](https://thebinaryhick.files.wordpress.com/2022/07/figure-21.png?w=812)Figure 21. Session’s key.

There is a second encryption key involved in this operation. It is stored in the file _network.loki.messenger_preferences.xml_ which is found in **~/files** in the app’s folder. See Figure 22.

![](https://thebinaryhick.files.wordpress.com/2022/07/figure-22.png?w=820)Figure 22. Encrypted database key with IV.

The area in the red box is the key to decrypt the database, but it has been encrypted (hence the XML tag **pref_database_encrypted_secret**) and encoded in base64. In the blue box is the initialization vector (IV) which is also encoded in base64. Using CyberChef, I decoded the key in the red box and converted it to hex give the value seen in Figure 23.

![](https://thebinaryhick.files.wordpress.com/2022/07/figure-23.png?w=820)Figure 23. Decoded database key.

This procedure merely decodes the key, but does not _decrypt_ it. Based on the information up to this point and what was seen in _KeyStoreHelper.java_, it appears the database key has been encrypted using AES-GCM. In order to decrypt the key we will need the input, the IV, and the GCM (or authentication) tag. As it turns out, the GCM tag is the last 32-bytes seen in the decoded value in Figure 23 (red box).

Now that all three components are obtained, the database key can be decrypted. See Figure 24.

![](https://thebinaryhick.files.wordpress.com/2022/07/figure-24.png?w=820)Figure 24. Putting it all together.

In Figure 24 the input is the blue box from Figure 23, the GCM tag is the red box from Figure 23, the IV is the blue box from Figure 22, and the Key is the red highlighted area from Figure 21. The 64-byte output is the key to decrypt _signal.db_ (red box in Figure 24). Remembering the PRAGMAs seen in Figure 15, we can apply those cipher settings in DB Browser for SQLite as “Custom.” Note that while the final output in Figure 24 is in hexadecimal, it should be applied as a “passphrase”. See Figure 25.

![](https://thebinaryhick.files.wordpress.com/2022/07/figure-25.png?w=820)Figure 25. Decrypted Session message content in signal.db (table sms).

So now we know Session is using the same method of encryption to protect its message content.

Messages are stored in the **sms** table but data about recipients and message threads are stored in the tables **session_contact_datbase** and **thread**, respectively. Group information is stored in the **group** table. The following SQL query will pull things together and order it by the time messages were _received_. In addition to 1-to-1 messages, this query will return group messages. Note that the query will also return calls, but I was unable to determine any value that delineated audio and video calls so calls are listed generically.

_SELECT  
datetime(sms.date_sent/1000,’unixepoch’) AS “Time Sent”,  
datetime(sms.date/1000,’unixepoch’) AS “Time Received”,  
session_contact_database.name AS “Other Party”,  
groups.title AS “Group Name”,  
CASE  
WHEN sms.type=2 THEN “Outgoing Call”  
WHEN sms.type=1 THEN “Incoming Call”  
WHEN sms.type=10485780 THEN “Received Message”  
WHEN sms.type=10485783 THEN “Sent Message”  
ELSE sms.type  
END AS “Message Type”,  
sms.body AS “Message”  
FROM  
sms  
JOIN thread ON thread._id=sms.thread_id  
LEFT JOIN session_contact_database ON session_contact_database.session_id=sms.address  
LEFT JOIN groups ON groups.group_id=thread.recipient_ids  
ORDER BY “Time Received” ASC_

The query output can be seen in Figure 26.

![](https://thebinaryhick.files.wordpress.com/2022/07/figure-26.png?w=820)Figure 26. SQL query results for messages.

The **sms** table does not contain message _replies_ (described further in the following section). So, the following is an alternate query:

_SELECT  
datetime(sms.date_sent/1000,’unixepoch’) AS “Time Sent”,  
datetime(sms.date/1000,’unixepoch’) AS “Time Received”,  
datetime(mms.date/1000,’unixepoch’) AS “Time Reply Received”,  
session_contact_database.name AS “Other Party”,  
groups.title AS “Group Name”,  
CASE  
WHEN sms.type=2 THEN “Outgoing Call”  
WHEN sms.type=1 THEN “Incoming Call”  
WHEN sms.type=10485780 THEN “Received Message”  
WHEN sms.type=10485783 THEN “Sent Message”  
ELSE sms.type  
END AS “Message Type”,  
sms.body AS “Message”,  
mms.body AS “Reply”,  
mms.quote_body AS “  
FROM  
sms  
JOIN thread ON thread._id=sms.thread_id  
LEFT JOIN session_contact_database ON session_contact_database.session_id=sms.address  
LEFT JOIN groups ON groups.group_id=thread.recipient_ids  
JOIN mms ON mms.quote_id=sms.date  
ORDER BY “Time Received” ASC_

The output from this alternate query should be verified as I did not have a data set available that included replies within a group chat.

Attachments…or not
------------------

Session, like Signal, stores files that are sent and received in the app in an encrypted format with a _.mms_ file extension. While the pertinent portion of the code in the Session Git repo _appears_ unchanged, I was unsuccessful in decrypting message attachments. I was able to recover the _modernKey_ (described in the previously linked paper) and associated _data_random_ values but I was unable to identify how those elements were used to decrypt the attachments using HMAC256 and AES-CTR. This was a major roadblock I was not able to overcome, unfortunately. The one thing I will note is that the key needed to decrypt attachments along with the IV is stored in the file _network.loki.messenger_preferences.xml_ under the tag **pref_attachment_encrypted_secret**. See Figure 27.

![](https://thebinaryhick.files.wordpress.com/2022/07/figure-27.png?w=820)Figure 27. Encrypted key and IV for attachments.

Figure 28 shows the location of attachments in the **~/app_parts** directory path.

![](https://thebinaryhick.files.wordpress.com/2022/07/figure-28-1.png?w=820)Figure 28. Attachments in app_parts.

Even with a lack of decrypted attachments, an examiner can still get attachment metadata from _signal.db_. Attachment metadata is stored in two tables: **parts** and **mms**. Additional data is found in the **thread**, **session_contact_database**, and **group** tables. Another SQL query will pull things together.

_SELECT  
datetime(mms.date_received/1000,’unixepoch’) AS “Time Sent”,  
datetime(mms.date/1000,’unixepoch’) AS “Time Received”,  
session_contact_database.name AS “Other Party”,  
groups.title AS “Group Name”,  
CASE  
WHEN mms.msg_box=10485780 THEN “Received Message”  
WHEN mms.msg_box=10485783 THEN “Sent Message”  
WHEN mms.msg_box=11010071 THEN “System Message”  
ELSE mms.msg_box  
END AS “Message Type”,  
part._data AS “Local Path To Attachment”,  
part.ct AS “Attachment Type”,  
part.data_size AS “Attachment Size”,  
mms.body AS “Message”,  
mms.quote_body AS “Message Replied To”,  
CASE  
WHEN datetime(mms.quote_id/1000,’unixepoch’)=”1970-01-01 00:00:00″ THEN “N/A”  
ELSE datetime(mms.quote_id/1000,’unixepoch’)  
END AS “Replied To Message Time”  
FROM  
mms  
LEFT JOIN part on part.mid=mms._id  
JOIN thread ON thread._id=mms.thread_id  
LEFT JOIN session_contact_database ON session_contact_database.session_id=mms.address  
LEFT JOIN groups on groups.group_id=thread.recipient_ids_

An interesting note about Session. Replies to messages are documented in the **mms** table, and not in the **sms** table with the rest of the text messages. So, message _replies_ will be returned in this query, to, in the event the alternate query for message content does not work. Output of the query is seen in Figure 29.

![](https://thebinaryhick.files.wordpress.com/2022/07/figure-29.png?w=820)Figure 29. Attachments. And replies?

The output includes the path to local storage, but it is different that was was shown in Figure 28. A representation of the path output is seen in Figure 30.

![](https://thebinaryhick.files.wordpress.com/2022/07/figure-30.png?w=820)Figure 30. Looks familiar.

The contents of this directory path mirror those seen in Figure 28.

A Few More Things
-----------------

Session’s settings are stored in the aforementioned _network.loki.messenger_preferences.xml_ file found in **~/shared_prefs**. There are three (3) values in the file, in addition to the ones already highlighted, that examiners should take care to note. See Figure 31.

![](https://thebinaryhick.files.wordpress.com/2022/07/figure-31.png?w=820)Figure 31. More values.

The first value, seen in the blue box, is the display name (XML tag **pref_profile_name**). This is the friendly name that is display to other users when chatting. This value can be changed by a user at any time. The second value, in the red box, is the Session ID for the phone (XML tag **pref_local_number**). This value cannot be changed by a user. However, if a user decides to uninstall the app and then re-install, a new Session ID is generated for the phone.

The last value, in the green box, is the encrypted key and IV for the log file, which is found in the **~/cache/log** directory path. The log file describes activities within the app. The process to decrypt the log file, however, is arduous. The file is encrypted in chunks, and each chunk has its own IV, length, and chunk of encrypted data. Based on my testing, it appears that each entry (i.e., each line) in the log file is encrypted separately from the rest. Again, examiners should read the paper previously referenced to understand the process.

Wrapping Up
-----------

Encrypted messaging apps are not going away. In fact, examiners can expect to encounter more as time carries on. So, it is helpful when examiners are able to research, understand a bit of code (pun sorta intended), and apply what they find to their examinations. This can help overcome obstacles they may face when tools don’t support a particular app.

Session is the new kid on the encrypted messaging block. It has been around for a couple of years now in its current form, and seems to be picking up steam, although it has a ways to go to get to the popularity level of Signal. It protects its communication using the Signal protocol. Because of this and available research, examiners may be able to uncover message content on the Android version of the app under certain circumstances.

Hopefully, someone will figure out how to decrypt attachments ;-). Until then, we will just have to settle for attachment metadata.