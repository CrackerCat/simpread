> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [citizenlab.ca](https://citizenlab.ca/2021/03/tiktok-vs-douyin-security-privacy-analysis/)

[Research![](https://citizenlab.ca/wp-content/themes/citizenlab-2.0.3/library/images/chevron-left.svg)](https://citizenlab.ca/category/research/)[App Privacy and Controls](https://citizenlab.ca/category/research/app-privacy-and-security/)

We covered some general questions on this report on our accompanying [FAQ](https://citizenlab.ca/2021/03/tiktok-and-douyin-explained/).

*   This report provides a comparative analysis of security, privacy, and censorship issues in TikTok and Douyin, both developed by ByteDance.
*   TikTok and Douyin do not appear to exhibit overtly malicious behavior similar to those exhibited by malware. We did not observe either app collecting contact lists, recording and sending photos, audio, videos or geolocation coordinates without user permission.
*   Despite not exhibiting overtly malicious behavior, Douyin contains features that raise privacy and security concerns, such as dynamic code loading and server-side search censorship. TikTok does not contain these features.
*   TikTok and Douyin’s Android apps share many parts of their source code. We postulate that ByteDance develops TikTok and Douyin starting out from a common code base and applies different customizations according to market needs. We observed that some of these customizations can be turned on or off by different server-returned configuration values. We are concerned but could not confirm that this capability may be used to turn on privacy-violating hidden features.
*   Both TikTok and Douyin have source code for restricting search results for content labeled as “hate speech,” “suicide prevention,” and “sensitive.” We suspect the “sensitive” field restriction refers to content that is “politically sensitive” but could not confirm.
*   The evidence we collected is inconclusive about whether TikTok employs political censorship of user posts. We did not test for post censorship on Douyin
*   Douyin restricts some political terms in search. TikTok did not restrict any of the keywords we tested.

App developers operating in China face unique challenges due to laws and regulations that hold companies accountable for the content published or transmitted on their platforms. Companies are expected to invest in human resources and technologies to moderate content and comply with government regulations on content controls. Companies which do not undertake such moderation and compliance activities can be fined or have their business licenses revoked

When Chinese app developers move beyond China and enter international markets, they must adapt their products to suit different user expectations and regulations while also maintaining their user base in China and compliance with Chinese law. This balancing act can lead to users inside and outside of China having vastly different experiences with an application. For example, WeChat, the most popular social platform in China, offers a single application that serves users around the world, but it [enables censorship for users based in China](https://citizenlab.ca/2016/11/wechat-china-censorship-one-app-two-systems/) to comply with local laws and regulations.

ByteDance, a China-based technology company develops TikTok, a video-based social media platform which is the first Chinese-made social media platform that reached global popularity, crossing [2 billion accumulated downloads in April 2020](https://sensortower.com/blog/tiktok-downloads-2-billion). The app started in China under the name Douyin, and was released as TikTok tailored for the international market. The two versions continue to be maintained for these separate markets. TikTok and Douyin are both for sharing short videos and have similar interfaces. However, they are entirely separate apps that have access to two separate platforms. Users on these two platforms cannot interact with each other. These platforms also have different registration processes, policies, and content.

TikTok’s huge popularity in North America in particular has put it under increased scrutiny especially following the attempt from the Trump Administration to ban the app in the U.S. due to alleged national security concerns. Despite TikTok’s popularity and prominent public discussions surrounding its security and privacy, neither TikTok nor Douyin have been thoroughly studied. The popularity and attention TikTok has gained poses key questions: Did ByteDance adapt its social media platform to the international market in alignment with industry norms? Or did it implement or retain some features that are required in the Chinese market that may present undesirable security and privacy risks to non-mainland China users?

These are complex questions that need to consider technical aspects and platform policies and practices. This report focuses on the technical characteristics of TikTok and Douyin through analysis of the source codes of TikTok and Douyin’s Android apps. The results of this analysis inform our understanding of how ByteDance develops the two apps for their respective markets.

During the course of this research, the [US](https://www.whitehouse.gov/presidential-actions/executive-order-addressing-threat-posed-tiktok/), [India](https://www.bbc.com/news/technology-53225720), [Pakistan](https://www.nytimes.com/2020/10/11/technology/tiktok-pakistan-ban.html), [Indonesia](https://www.dw.com/en/indonesia-blocks-pornographic-tik-tok-app/a-44537230), and [Bangladesh](https://www.dw.com/en/bangladesh-anti-porn-war-bans-blogs-and-google-books/a-47684058) have put forth government enforcements to restrict or outright ban the usage of TikTok, often out of “national security” concerns. These bans centre on concerns that because TikTok’s parent company ByteDance is headquartered in Beijing, the Chinese government could compel it to share personal data of foreign nationals. Although this type of cooperation is a possible scenario, little research has been done to understand how and to what degree the Chinese government dictates control over TikTok’s operation. The steps taken by governments to ban or restrict access to TikTok for national security reasons have numerous political motivations. Our aim is to undertake an objective and evidence-based technical analysis of TikTok from the perspective of user privacy and security. Such an assessment has not been done thoroughly, and yet should be an essential prerequisite to any informed public policy towards use of the application.

The report is grouped into the following sections:

[**Methodology**](#methodology "Methodology")

This section describes our methods for analyzing the Android versions of TikTok and Douyin.

[**Background**](#background)

This section provides a high level comparison of TikTok and Douyin and shows that TikTok has two regional versions: one for East Asian and Southeast Asian countries; the other for all other countries. These two versions are almost identical in terms of interface and code base. Users can use either of these apps to access the same global platform. However they differ in some aspects, such as regional law compliance.

[**Privacy**](#privacy "Privacy")

This section reports results from traffic analysis and shows what data the applications collect and when they are transmitted, and the destinations they are sent to. Overall we find that neither app appears to collect sensitive personal data (e.g., contact lists, user files, or geolocation coordinates) without permission. Douyin contains many features unsuitable for the international market, such as dynamic code loading and server side search censorship. TikTok does not contain these features. TikTok collects a large variety of device information and usage pattern information. It also makes use of third-party advertisement and tracking services, including Facebook, Firebase Analytics, and AppsFlyer. These characteristics are not exceptional when compared to industry norms.

[**Security**](#security "Security")

This section reports our results from source code analysis which finds that TikTok and Douyin do not appear to exhibit overtly malicious behavior. We did not observe either app collecting contact lists, recording and / or sending photos, audio, videos, or geolocation coordinates without user permission. Douyin contains many features that users may find undesirable in the international market, such as dynamic code loading and server side search censorship. TikTok does not contain these features.

[**Censorship**](#censorship "Censorship")

This section reports results from our analysis of post and search censorship on TikTok and Douyin. Based on our results it is unclear if TikTok employs political censorship of user posts. We did not test for post censorship on Douyin. Both TikTok and Douyin have source code for server-side search censorship, which is capable of restricting search results for content labeled as “hate speech,” “suicide prevention,” and “sensitive.” We suspect the “sensitive” restriction refers to content that is “politically sensitive” but could not confirm. Douyin restricts some political terms in search. TikTok did not restrict any of the keywords we tested.

In this section, we provide a high-level overview of our methodology. Further details on our methods are described in respective sections.

### Source code examination motivated by captured network traffic

We used the following process in our analysis of captured network traffic.

1.  Capture network traffic when using the app normally.
2.  From captured traffic, identify interesting data, such as personal information, device identifiers, program executables, and encrypted strings.
3.  Search the source code for field names in those network requests to determine how the data were collected or generated. When exploring the source code this way, we can often also find code concerning privacy and security.

This approach allows us to quickly identify interesting parts of the code instead of needing to go through the complete code base.

### Web scraping and data analysis

We used web scraping methods to discover platform censorship in the form of content (post) takedowns. We built two web scrapers: one to collect existing content and another to check if collected content was still available on the platform.

### Automated user operation of the app

After we discovered that TikTok and Douyin both include capabilities to perform server-side censorship of platform search results, we explored if they actually filter content and, if so, what specific content is filtered.

To probe these questions, we developed a program that controls the app to perform searches on the platform simulating routine user-performed searches. We execute this program iteratively, performing searches on keyword combinations censored by another Chinese platform, WeChat, as found [in prior research](https://citizenlab.ca/2016/11/wechat-china-censorship-one-app-two-systems/). Finally, we compare the search results returned by TikTok and Douyin.

### Traffic Interception, and Analysis Tool Setup

Our traffic interception and analysis setup includes the following components:

*   System rooted with Magisk
*   Magisk modules used:
    *   Always Trust User Certificates v0.4 from [giacomoferretti’s fork](https://github.com/giacomoferretti/MagiskTrustUserCerts/releases/tag/v0.4)
    *   MagiskFrida 12.8.17-1
    *   Riru – Core v19.7
    *   Riru – EdXposed v0.4.5.1_beta(4463) (YAHFA)
*   [EdXposed](https://github.com/ElderDrivers/EdXposed) modules used:
    *   TrustMeAlready (com.virb3.trustmealready) 1.11
    *   XPrivacyLua (eu.faircode.xlua) 1.27
*   Installed intercepting server’s SSL root CA on testing phone
*   Test phone shares WiFi local network with intercepting server, configure test phone to use intercepting server as system wide proxy
*   Intercepting server: Burp Suite Community Edition v2020.1

### Testing Environment

*   Android 9 (LineageOS 16)
*   System locale set to Traditional Chinese (Taiwan)

### App Version Numbers Under Analysis

Even within a single app behaviors may differ across different versions. Hence, we document the precise app versions numbers of the apps we analyzed. We obtained these versions in November 2019 when we started the research. Precise version numbers are specified within the “AndroidManifest.xml” file contained within every Android app package (APK).

*   Musically[1](#fn1): android:versionCode=”2021407050″ android:versionName=”14.7.5″
*   Trill: android:versionCode=”963″ android:versionName=”9.6.3”
*   Douyin: android:versionCode=”960″ android:versionName=”9.6.0″

The value “versionName” is displayed on the Google Play Store. It is end-user facing but not precise, because the Google Play Store allows developers to upload a new version of an app without changing the versionName. Therefore, we also specified “versionCode,” which is used [internally within the Android operating system](https://developer.android.com/studio/publish/versioning#appversioning) and Google Play Store to differentiate between versions, and is required to be a strictly-increasing integer across app updates.

### Obtaining App Package Files (APK)

Reverse engineering an Android app starts with obtaining the Android Package file (APK) for the app. Commonly, researchers will download these files from third-party APK hosting sites like [APKMirror](https://www.apkmirror.com/) and [Apkpure](https://apkpure.com/). These sites serve users who do not have access to the Google Play Store, or want to bypass its downloading restrictions. Files on these sites are either first obtained by the site itself (Apkpure) or submitted by voluntary users (APKMirror). Since the files have gone through the hands of third parties, there is a higher chance of them being tampered with.

Alternatively, researchers can install the app normally on their phone using the Google Play Store then export the APK files using tools like `adb`. This approach guarantees the integrity of the files but does not allow researchers to download apps which are unavailable for their hardware or region.

Here, we utilized an open source Google Play Store client “[Aurora Store](https://gitlab.com/AuroraOSS/AuroraStore).”

Aurora Store offers the following features that aids our work on reverse engineering:

*   Download and install apps from Google Play Store.
*   Save downloaded APK to specified path.
*   Offers an “anonymous account” which all users can access, so users do not need to use their own Google account.
*   Switching to different countries’ Play Store simply by using a VPN to change IP address. After switching, users can perform searches, view top lists, and download apps, as if they are in that country.
*   Specify older APK versions to download. However, not all historical versions are available. Availability is dependent on which versions the Play Store still hosts.
*   Ability to fake device profiles. This feature allows users to download APK variants (CPU architecture, screen resolution, locale, etc.) that were not meant for their operating device.

Limitations
-----------

The source code of Douyin and TikTok is vast, and so are the differences between them. Given time constraints, we could not examine every source code component and test the apps in every circumstance, which means our methods could not find every security issue, privacy violation, and censorship event. However, our methodology should lead us to find overt issues in these areas if they existed. This research only focuses on TikTok and Douyin’s Android versions and does not analyze iOS versions of the apps.

In this section we review related research and present high level overviews of Douyin and TikTok features and characteristics.

Previous work on security and privacy issues with TikTok provide useful background information and baseline technical analyses that inform our research. This section reviews this prior work and how our research contributes to understanding TikTok and Douyin.

In December 2019, Matthias Eberl, a German journalist, conducted a [technical analysis](https://rufposten.de/blog/2019/12/05/privacy-analysis-of-tiktoks-app-and-website/) on TikTok’s app and website. They found that TikTok’s data collection using third-party trackers was in apparent conflict with the GDPR (General Data Protection Regulation). Ebrel discovered the TikTok website identifies users using “canvas fingerprinting,” a fingerprinting technique in which the website asks the browser to draw a hidden image, and using that unique image to identify the browser version, operating system, and other information regarding the execution environment.

In August 2020, a French security researcher, Baptiste Robert conducted [technical analysis](https://medium.com/@fs0c131y/tiktok-logs-logs-logs-e93e8162647a) on network traffic transmitted by the TikTok Android app. They focused on what, when, and where TikTok sends its data, concluding that TikTok did not have suspicious behavior and was not exfiltrating unusual data, which coincides with our findings. They also identified the “ttEncrypt” custom encryption algorithm, but did not explore what data this contained. Our research fills this gap by presenting a way to capture the original data before the encryption and analyzing its contents.

In September 2020, the Australian Strategic Policy Institute (ASPI) published a report on TikTok and WeChat’s censorship, privacy concerns, and data collection titled [TikTok and WeChat: Curating and controlling global information flows](https://www.aspi.org.au/report/tiktok-wechat). ASPI summarized from previous research multiple themes that were censored by TikTok, and also uncovered new ones. Those themes included world leaders, politics, protests, LGBTIQ+ issues, and Xinjiang. ASPI’s approaches included code inspection and posting content with censored hashtags. Our approach was to track the availability of posts with sensitive hashtags. The evidence we found was inconclusive as to whether TikTok employed any censorship of user posts. However, though inconclusive, our evidence does not contradict ASPI’s findings.

ASPI also looked into TikTok’s privacy concerns and data collection using methods similar to ours. They did not observe TikTok’s Android app carrying out any overtly malicious activity akin to spyware, which aligns with our findings.

ASPI’s research speculated that the TikTok Android app could collect keystroke data and suggested it may be obscured by the app’s custom encryption. However, the researchers were not able to decrypt the encryption algorithm and could not confirm if the collection was taking place. Assuming the “custom encryption” they mentioned is what we found and called “ttEncrypt” (see section below), we can confirm that keystrokes were not encrypted and sent using ttEncrypt. We did not conduct further research into whether TikTok collected keystroke data.

An investigative report by [The Intercept](https://theintercept.com/2020/03/16/tiktok-app-moderators-users-discrimination/) published in March 2020, revealed through leaked Chinese policy documents that TikTok moderators were instructed to suppress content posted by users deemed too “ugly”, “poor” or “disabled” for the platform. The documents also show instructions to moderators to block livestreams with political content that harmed “national honor” or national security. A TikTok spokesperson claimed these were old documents and these policies were “either no longer in use, or in some cases appear to never have been in place”.

A September 2020 [academic article](https://policyreview.info/articles/analysis/going-global-comparing-chinese-mobile-applications-data-and-user-privacy) by Lianrui Jia and Lotus Ruan examined and compared data and privacy governance by four China-based mobile applications and their international versions, including Douyin and its international version TikTok. They employed the walkthrough method to test these apps in actual operation, as well as conducting a content analysis of the privacy policies and terms of service of each mobile app.

Our research complements Jia and Ruan’s article by providing insights into the technical aspects of the applications. Our findings on the two TikTok Android versions available in Asian and non-Asian app stores and their differences corresponds to Jia and Ruan’s finding that TikTok/Douyin are more attentive to users from different geographical regions by designating jurisdiction-specific privacy policies and terms of service.

Their study also showed that the Chinese domestic and international versions of the same app vary in data and privacy protection standards. However, as they also noted, there can be inconsistencies between what the app policy states and what the code of the app appears to do. Our research closes this gap by providing technical evidence showing the different data collection practices from TikTok and Douyin.

TikTok’s two regional versions
------------------------------

When obtaining the APK files for TikTok, we noticed that there are actually two versions of the app available in 1) East and Southeast Asian and 2) other remaining countries’ Play Stores. The App IDs of these apps are “com.ss.android.ugc.trill” and “com.zhiliaoapp.musically” respectively. We will use “Trill” and “Musically” to refer to them for short in the rest of the report. Note that, there was a company named “Musical.ly,” which was acquired by ByteDance in 2017. We will use “Musically” to refer to the TikTok app variant, and “Musical.ly” to refer to the company and its product before acquisition.

Their differences compared in this table, with Douyin listed for reference:

<table headertable="true"><thead><tr><th>Version</th><th>Trill</th><th>Musically</th><th>Douyin</th></tr></thead><tbody><tr><td>App ID</td><td>com.ss.android.ugc.trill</td><td>com.zhiliaoapp.musically</td><td>com.ss.android.ugc.aweme</td></tr><tr><td>Publisher (as listed on Google Play Store)</td><td>TikTok Pte (Singapore)</td><td><a href="https://web.archive.org/web/20201014025432/https://play.google.com/store/apps/details?id=com.zhiliaoapp.musically"><u>TikTok Inc (USA)</u></a>, but was changed to <a href="https://web.archive.org/web/20201021055608/https://play.google.com/store/apps/details?id=com.zhiliaoapp.musically"><u>TikTok Pte in October 2020</u></a></td><td>Not listed on Google Play Store</td></tr><tr><td>Platform policies</td><td colspan="2"><a href="https://www.tiktok.com/legal/terms-of-use?lang=en"><u>Terms of Service</u></a> and <a href="https://www.tiktok.com/legal/privacy-policy?lang=en"><u>Privacy Policy</u></a></td><td><a href="https://www.douyin.com/agreements/?id=6773906068725565448"><u>User Agreement</u></a> and <a href="https://www.douyin.com/agreements/?id=6773901168964798477"><u>Privacy Policy</u></a></td></tr><tr><td>Available countries, tested on June 10, 2020, in ISO country codes, countries not listed are not tested.</td><td><ul><li>KR</li><li>PH</li><li>TW</li><li>TH</li><li>VN</li></ul></td><td><ul><li>BD</li><li>BO</li><li>BR</li><li>CA</li><li>FR</li><li>DE</li><li>GR</li><li>HU</li><li>IN</li><li>LY</li><li>MX</li><li>NP</li><li>NL</li><li>NG</li><li>PE</li><li>RU</li><li>RS</li><li>CH</li><li>US</li></ul></td><td>Intended for Chinese users, but does not forbid international users from downloading the APK from their website.</td></tr><tr><td>User count</td><td>100,000,000+ Play Store downloads (November 2020)</td><td>1,000,000,000+ Play Store downloads (November 2020)</td><td><a href="https://mp.weixin.qq.com/s/mjzr2ssMpmDdVeMeiOTb3g"><u>Over 400 million daily active users (January 2020)</u></a></td></tr><tr><td>Server domain examples</td><td><ul><li>log.tiktokv.com</li><li>api.tiktokv.com</li></ul></td><td><ul><li>log2.musical.ly</li><li>api2.musical.ly</li></ul></td><td><ul><li>log.snssdk.com</li><li>aweme.snssdk.com</li></ul></td></tr><tr><td>“Discover” tab<a role="doc-noteref" href="#fn2" data-footnote-backlink-ref="fnref2" data-footnote-ref="#fn2"><sup>2</sup></a></td><td>Mostly Taiwanese posts</td><td>Mostly Canadian posts</td><td>(Not compared)</td></tr></tbody></table>

Table 1: Basic comparison of Musically, Trill, and Douyin.

Their permissions:

<table headertable="true"><thead><tr><th>Permission</th><th>Trill</th><th>Musically</th><th>Douyin</th></tr></thead><tbody><tr><td>ACCESS_COARSE_LOCATION<a role="doc-noteref" href="#fn3" data-footnote-backlink-ref="fnref3" data-footnote-ref="#fn3"><sup>3</sup></a></td><td>✓</td><td></td><td>✓</td></tr><tr><td>ACCESS_FINE_LOCATION</td><td>✓</td><td></td><td>✓</td></tr><tr><td>ACCESS_LOCATION_EXTRA_COMMANDS</td><td></td><td></td><td>✓</td></tr><tr><td>ACCESS_NETWORK_STATE</td><td>✓</td><td>✓</td><td>✓</td></tr><tr><td>ACCESS_WIFI_STATE</td><td>✓</td><td>✓</td><td>✓</td></tr><tr><td>AUTHENTICATE_ACCOUNTS</td><td>✓</td><td>✓</td><td>✓</td></tr><tr><td>BLUETOOTH</td><td></td><td></td><td>✓</td></tr><tr><td>BLUETOOTH_ADMIN</td><td></td><td></td><td>✓</td></tr><tr><td>BROADCAST_PACKAGE_ADDED</td><td></td><td></td><td>✓</td></tr><tr><td>BROADCAST_PACKAGE_CHANGED</td><td></td><td></td><td>✓</td></tr><tr><td>BROADCAST_PACKAGE_INSTALL</td><td></td><td></td><td>✓</td></tr><tr><td>BROADCAST_PACKAGE_REPLACED</td><td></td><td></td><td>✓</td></tr><tr><td>CAMERA</td><td>✓</td><td>✓</td><td>✓</td></tr><tr><td>CHANGE_CONFIGURATION</td><td></td><td></td><td>✓</td></tr><tr><td>CHANGE_NETWORK_STATE</td><td></td><td></td><td>✓</td></tr><tr><td>CHANGE_WIFI_STATE</td><td></td><td></td><td>✓</td></tr><tr><td>EXPAND_STATUS_BAR</td><td>✓</td><td>✓</td><td>✓</td></tr><tr><td>FLASHLIGHT</td><td>✓</td><td>✓</td><td>✓</td></tr><tr><td>FOREGROUND_SERVICE</td><td>✓</td><td>✓</td><td>✓</td></tr><tr><td>GET_TASKS</td><td>✓</td><td>✓</td><td>✓</td></tr><tr><td>INSTALL_SHORTCUT</td><td></td><td></td><td>✓</td></tr><tr><td>INTERNET</td><td>✓</td><td>✓</td><td>✓</td></tr><tr><td>MANAGE_ACCOUNTS</td><td>✓</td><td>✓</td><td></td></tr><tr><td>MODIFY_AUDIO_SETTINGS</td><td>✓</td><td>✓</td><td>✓</td></tr><tr><td>READ_APP_BADGE</td><td>✓</td><td>✓</td><td>✓</td></tr><tr><td>READ_CONTACTS</td><td>✓</td><td>✓</td><td>✓</td></tr><tr><td>READ_EXTERNAL_STORAGE</td><td>✓</td><td>✓</td><td>✓</td></tr><tr><td>READ_PHONE_STATE</td><td>✓</td><td></td><td>✓</td></tr><tr><td>RECEIVE_BOOT_COMPLETED</td><td>✓</td><td>✓</td><td>✓</td></tr><tr><td>RECORD_AUDIO</td><td>✓</td><td>✓</td><td>✓</td></tr><tr><td>REORDER_TASKS</td><td>✓</td><td>✓</td><td>✓</td></tr><tr><td>REQUEST_INSTALL_PACKAGES</td><td>✓</td><td>✓</td><td>✓</td></tr><tr><td>RESTART_PACKAGES</td><td></td><td></td><td>✓</td></tr><tr><td>SUBSTITUTE_NOTIFICATION_APP_NAME</td><td></td><td></td><td>✓</td></tr><tr><td>SYSTEM_ALERT_WINDOW</td><td></td><td></td><td>✓</td></tr><tr><td>UNINSTALL_SHORTCUT</td><td></td><td></td><td>✓</td></tr><tr><td>UPDATE_APP_BADGE</td><td>✓</td><td>✓</td><td>✓</td></tr><tr><td>USE_CREDENTIALS</td><td>✓</td><td>✓</td><td></td></tr><tr><td>VIBRATE</td><td>✓</td><td>✓</td><td>✓</td></tr><tr><td>WAKE_LOCK</td><td>✓</td><td>✓</td><td>✓</td></tr><tr><td>WRITE_EXTERNAL_STORAGE</td><td>✓</td><td>✓</td><td>✓</td></tr><tr><td>WRITE_SETTINGS</td><td></td><td></td><td>✓</td></tr><tr><td>WRITE_SYNC_SETTINGS</td><td>✓</td><td>✓</td><td>✓</td></tr></tbody></table>

Table 2: Comparison of Musically, Trill, and Douyin’s permissions.

What is the reason behind having two versions? According to APKMirror’s history, Musically was available as early as [January 1, 2017](https://www.apkmirror.com/apk/tiktok-inc/tik-tok-including-musical-ly/musical-ly-5-0-2-release/musical-ly-5-0-2-android-apk-download/) and Trill was available as early as [June 29, 2018.](https://www.apkmirror.com/apk/tiktok-pte-ltd/tik-tok/tik-tok-2-4-7-release/tik-tok-2-4-7-android-apk-download/) [On August 2, 2018,](https://en.wikipedia.org/wiki/Musical.ly#Merger_into_TikTok) Bytedance consolidated the user accounts of Musical.ly and TikTok, merging the two apps into one and keeping the title TikTok. Musical.ly’s merger into Bytedance was announced on [November 3, 2017](https://web.archive.org/web/20181019163943/https://www.bloomberg.com/research/stocks/private/snapshot.asp?privcapid=316026378). This timeline means that, before August 2, 2018, there was a period of time when both Musically and Trill were available in the Google Play Store (though potentially not in every country). It is likely that both apps already accumulated their own user base, and after the merger it was easier to simply upgrade both apps to the new merged-code version, instead of asking users to install another app.

Other than the reasons above, this version distinction is also used to adjust interfaces and provide user settings tailored to the targeted regions. Users are only given the ability to opt-out of ad personalization in Musically, which is likely due to the requirements of the European General Data Protection Regulation (GDPR).

[![](https://citizenlab.ca/wp-content/uploads/2021/03/img1.png)](https://citizenlab.ca/wp-content/uploads/2021/03/img1.png)

Figure 1: Musically offers the option to opt-out of ad personalization

[![](https://citizenlab.ca/wp-content/uploads/2021/03/img2.png)](https://citizenlab.ca/wp-content/uploads/2021/03/img2.png)

Figure 2: Trill, under the same settings page, does not offer ad personalization option.

Note that, **in this report comparing TikTok and Douyin, we say “TikTok” to refer to the Musically variant unless otherwise stated.**

Code Sharing Among Musically, Trill, and Douyin
-----------------------------------------------

When exploring Musically, Trill, and Douyin’s source code, we found that many of the functions and classes were identical (implemented in the same code). Musically, Trill, and Douyin also share many code packages with identical names and paths, for instance:

*   `com.ss.android.ugc.aweme`
*   `com.bytedance.frameworks`
*   `com.bytedance.ttnet`

There were many more instances of identical code packages, however, we are uncertain exactly to what degree the three variants share common code, as their code bases are vast. For the parts which we have examined, the differences between Musically and Trill are fewer than the differences between Douyin and the other two. This is expected because Douyin serves a China-only platform separate from the global platform served by regional variants Trill and Musically.

In the rest of this section, we show examples of how hard-coded configuration values and server-returned configuration values can change the app’s behavior.

### Hard-coded configuration values

In some cases, differences in Musically, Trill, and Douyin’s behavior are determined by internal configuration values instead of source code differences. In this section, we present a relatively simple example of how internal configuration values affected which logging servers the apps used.

Musically includes a function `com.bytedance.ies.ugc.statisticlogger.config.C9656b.m28354a`[4](#fn4) to choose its log server:

[![](https://citizenlab.ca/wp-content/uploads/2021/03/img3.png)](https://citizenlab.ca/wp-content/uploads/2021/03/img3.png)

Figure 3: Musically includes a function m28354a, which chooses the log server.

In this function we see a switch: depending on the return value of C9593b.m28186n(), the log server will be set to log.tiktokv.com (used by Trill in practice) or log2.musical.ly (used by Musically in practice).

What is this function “C9593b.m28186n()”? From Douyin’s source code, we see calls to the unobfuscated name of “C9593b.m28186n()” present in the same location of the source code:

[![](https://citizenlab.ca/wp-content/uploads/2021/03/img4.png)](https://citizenlab.ca/wp-content/uploads/2021/03/img4.png)

Figure 4: At the same location of the source code, Douyin uses AppContextManager.INSTANCE.getClientType() in place of C9593b.m28186n() in Musically.

Therefore, we can see it is actually the return value of AppContextManager.INSTANCE.getClientType() that determines which log server will be used. From this name, we can infer that the value returned by this function represents the type of client.

How is getClientType() implemented, then? This is getClientType()’s definition in Douyin:

[![](https://citizenlab.ca/wp-content/uploads/2021/03/img5.png)](https://citizenlab.ca/wp-content/uploads/2021/03/img5.png)

Figure 5: Implementation of getCLientType() in Douyin.

The field clientType is assigned only in one place: com.bytedance.ies.ugc.appcontext.AppContextManager.init() :

[![](https://citizenlab.ca/wp-content/uploads/2021/03/img6.png)](https://citizenlab.ca/wp-content/uploads/2021/03/img6.png)

Figure 6: The field clientType is assigned only in com.bytedance.ies.ugc.appcontext.AppContextManager.init().

clientType is assigned initially from the value of bVar.f9093l. bVar.f9093l also only ever has one assignment, in com.p232ss.android.ugc.aweme.app.launch.C3888b:

[![](https://citizenlab.ca/wp-content/uploads/2021/03/img7.png)](https://citizenlab.ca/wp-content/uploads/2021/03/img7.png)

Figure 7: clientType is assigned with the value of bVar.f9093l. bVar.f9093l is assigned only in com.p232ss.android.ugc.aweme.app.launch.C3888b.

From here we can see, in Douyin, the value i is initially assigned 0; later there is a isSupport check to potentially override the value from hot patches (see more details in the Douyin Hot Patching section below).

Going back to Musically’s source code, we see a similar process to assign clientType:

Definition of C9593b.m28186n():

[![](https://citizenlab.ca/wp-content/uploads/2021/03/img8.png)](https://citizenlab.ca/wp-content/uploads/2021/03/img8.png)

Figure 8: Definition of C9593b.m28186n(), which returns the value of f27341o.

Where f27341o is assigned:

[![](https://citizenlab.ca/wp-content/uploads/2021/03/img9.png)](https://citizenlab.ca/wp-content/uploads/2021/03/img9.png)

Figure 9: f27341o is assigned with the value of cVar.f27366l.

Where f27366l is assigned:

[![](https://citizenlab.ca/wp-content/uploads/2021/03/img10.png)](https://citizenlab.ca/wp-content/uploads/2021/03/img10.png)

Figure 10: f27366l is assigned with the value of C22024c.m72882a().

Where m72882a() is defined:

[![](https://citizenlab.ca/wp-content/uploads/2021/03/img11.png)](https://citizenlab.ca/wp-content/uploads/2021/03/img11.png)

Figure 11: Definition of m72882a(), which returns a fixed value “2.”

Therefore we can see, for Musically, the returned clientType value is 2, so it will execute AppLog.setHostI(“log2.musical.ly”); . Douyin returns a clientType value as 0, so none of the switch conditions will be executed. We have also verified that Trill returns a clientType value of 1, through identically-implemented functions as Musically.

This instance of code sharing is one of the more basic examples we found. There are many other variant (Musically, Trill, or Douyin) configurations, checked in various places in the source code, to determine variant-specific behaviors.

Many of these variables are defined statically within the source code and can be read by calling relevant functions in the class com.bytedance.ies.ugc.appcontext.AppContextManager in Douyin. There are functionally equivalent classes in Trill and Musically but with their names obfuscated, so we only show the Douyin version here.

In the AppContextManager class, we can see other functions that can tell variant differences. We list some of their names and explanations below (All functions discussed here also exist in Trill and Musically; we only list the names from Douyin because they are unobfuscated.)

<table headertable="true"><thead><tr><th>Function name</th><th>Return value in Douyin</th><th>Return value in Trill</th><th>Return value in Musically</th><th>Notes</th></tr></thead><tbody><tr><td><code>getAppId</code></td><td>1128</td><td>1180</td><td>1233</td><td>This value show up as the “aid” field in some HTTP API calls</td></tr><tr><td><code>isDouyinLite</code></td><td>false</td><td>false</td><td>false</td><td></td></tr><tr><td><code>getPushScheme</code></td><td>(empty string)</td><td>snssdk1180</td><td>musically</td><td>Probably related with setting the push notification scheme</td></tr><tr><td><code>isCN</code></td><td>true</td><td>false</td><td>false</td><td></td></tr><tr><td><code>isI18n</code></td><td>false</td><td>true</td><td>true</td><td></td></tr><tr><td><code>isMusically</code></td><td>false</td><td>false</td><td>true</td><td></td></tr></tbody></table>

Table 3: Functions and their return values that determine different behaviors in Douyin, Trill, and Musically.

In AppContextManager, there is a special set of variant variables with their sources not statically defined in the source code, but rather embedded in a hidden location within the APK file. To read these variables, Douyin and TikTok include a special “ApkUtil” component to directly open the installed APK file (usually present under /data/app), seek a special file location, then decode the values from there.

Under Google’s APK signature scheme v2 (since Android 7), custom fields are added in the APK signing block to store these values. These values are not protected by the APK’s check from tampering, therefore they are useful for non-critical information to be stored after the APK is signed. TikTok and Douyin are very likely using a tool developed by Meituan (a Chinese company) called Walle to achieve this feature. Information about Walle can be found in [Meituan’s engineering blog](https://tech.meituan.com/2017/01/13/android-apk-v2-signature-scheme.html).

Below is the list of variables and examples of how they are used. Note that the list may not be complete as we only look through which fields are accessed from the source code. There may be other unused fields present in the APK.

<table headertable="true"><thead><tr><th>Name</th><th>Usage example</th></tr></thead><tbody><tr><td><code>GIT_SHA</code></td><td></td></tr><tr><td><code>GIT_BRANCH</code></td><td></td></tr><tr><td><code>release_build</code></td><td>The value is sent in some telemetry messages.</td></tr><tr><td><code>meta_umeng_channel</code></td><td>If set to “local_test” (not the case for all 3 of our tested APK), some debugging functions are enabled.</td></tr></tbody></table>

Table 4: List of hidden variables embedded within Douyin and TikTok’s APK file.

### Server-returned configuration values

When a user logs in, Musically sends a request towards [https://api2-16-h2.musical.ly/aweme/v1/abtest/param/](https://api2-16-h2.musical.ly/aweme/v1/abtest/param/), which returns a list of [A/B testing](https://en.wikipedia.org/wiki/A/B_testing) parameters. Here, we look at a specific parameter “facebook_story”. Searching for “facebook_story” led us to `com.p420ss.android.ugc.aweme.setting.model.AbTestModel`:

[![](https://citizenlab.ca/wp-content/uploads/2021/03/img12.png)](https://citizenlab.ca/wp-content/uploads/2021/03/img12.png)

Figure 12: The AbTestModel class defines many A/B testing parameters

Judging from its label “facebook story 分享开 (facebook story sharing on)” and “facebook story 分享关 (facebook story sharing off)”, the parameter seems to control whether to enable sharing TikTok videos to Facebook Story. There is a function `com.p420ss.android.ugc.aweme.setting.C37101d.mo93517z()` that wraps around the parameter:

[![](https://citizenlab.ca/wp-content/uploads/2021/03/img13.png)](https://citizenlab.ca/wp-content/uploads/2021/03/img13.png)

Figure 13: The function mo93517z() returns the value of the isFacebookStoryEnable parameter.

Finally, the function is called from the another function com.p420ss.android.ugc.aweme.share.C37481at.m120061a(). If mo93517z() returned true, m120061a() appeared to construct the sharing information for Facebook Story.

[![](https://citizenlab.ca/wp-content/uploads/2021/03/img14.png)](https://citizenlab.ca/wp-content/uploads/2021/03/img14.png)

Figure 14: If mo93517z() returned true, m120061a() appeared to construct the sharing information for Facebook Story

### Other A/B testing parameters

Musically’s AbTestModel class contains many other A/B testing parameters, some of which mention “抖音” (Douyin) in their labels:

[![](https://citizenlab.ca/wp-content/uploads/2021/03/img15.png)](https://citizenlab.ca/wp-content/uploads/2021/03/img15.png)

Figure 15: Douyin-specific A/B testing parameters present in Musically.

Musically does not check for this parameter anywhere in its source code. However for Douyin, its AbTestModel class contains the same parameter. It also checks for the parameter:

[![](https://citizenlab.ca/wp-content/uploads/2021/03/img16.png)](https://citizenlab.ca/wp-content/uploads/2021/03/img16.png)

Figure 16: Douyin uses the douyinLoginWhiteInterface A/B testing parameter.

This is another case where TikTok and Douyin share their source code. In this case the Douyin-specific parameter does not affect Musically’s behavior at all because the code that checks this parameter was removed. However, for unknown reasons, the parameter variable itself is preserved (figuratively, this parameter variable is like a button on a device that does not do anything when pressed) There are many other similar cases of Douyin-specific parameters and code present in TikTok.

In the small portion of code which we had examined, we did not find any case in which undesirable features could be enabled by server-returned configuration values. However, we are still concerned that this dormant code originally meant for Douyin may be activated in TikTok accidentally, or even intentionally.

A privacy concern for mobile applications is user data being collected and transmitted through networks without the knowledge of users.

By intercepting an application’s network traffic, we can learn what data the application is collecting, when it is transmitted, and its destination. Intercepting application network traffic requires a specific testing environment setup, which we documented above in the Methodology section.

With our network traffic interception setup, we operate the apps in a sequence that mimics how a typical user would use the app using the following steps:

1.  Launch the app. (The app’s storage was wiped before launch, to restore it to a pristine state in which the app thinks it was launching for the first time after installation.)
2.  Deny all runtime permission requests if they were prompted.
3.  Scroll through the operation tutorial phase until the main video post feed was displayed.
4.  Scroll through the first few videos in the feed.

We then inspect the network traffic that was recorded during our operation, specifically looking for:

1.  Device information, such as phone numbers and MAC addresses.
2.  Personal information, such as contact list, photos, and files stored on the device.

Our experiments produced the following high-level findings:

*   Neither Douyin nor TikTok (Musically) seemed to upload contact lists, photos, or user files.
*   In terms of first-party data collection, Douyin and Musically collects a similar set of device information, with Douyin collecting slightly more.
*   In terms of third-party data collection, the difference between Douyin and TikTok is that they use different sets of third-party service providers. Douyin uses mostly Chinese services, while TikTok uses global services.
*   The data items collected by Douyin include MAC addresses, which allows highly precise identification of an individual device. TikTok did not collect MAC addresses.
*   While TikTok collected many data items, overall they still fall within general industry norms for user data collection.

In the subsections below, we document and analyze in detail the data collected by TikTok and Douyin.

### Advertisement and Tracking Services

Like most apps on the market, Douyin and TikTok include first-party trackers (that send information to app developers themselves) and third-party trackers (that send information to other companies). Usually, these trackers collect device information mainly for advertisement targeting, telemetry, debugging, and anti-abuse purposes. Depending on the granularity, this information may allow precise identification of individual users and profiling of them.[5](#fn5)

To examine what information was sent, we set up a test phone and network traffic interceptor and operated the app following the steps outlined above.

In this section, we list the trackers and the sensitive information that were sent. We only list the information that we have high confidence in identifying. Some device-identifying data digits and characters are redacted with the character “x.” During our tests we observed other fields being sent but could not conclusively identify the meaning of the content.

#### Douyin

##### Third-party tracking

Traffic captured on Feb 21, 2020

<table headertable="true"><thead><tr><th colspan="2">Aliyun Mobile Analytics API</th></tr></thead><tbody><tr><td>Request endpoint</td><td>Sensitive information sent</td></tr><tr><td>http://adash.man.aliyuncs.com/man/api?ak=23356390&amp;s=09e071959952d6d39b1af851829e1d5fa70e93ae</td><td>None</td></tr></tbody></table><table headertable="true"><thead><tr><th colspan="2">amdcopen.m.taobao.com</th></tr></thead><tbody><tr><td>Request endpoint</td><td>Sensitive information sent</td></tr><tr><td>http://amdcopen.m.taobao.com/amdc/mobileDispatch?appkey=umeng%3A57bfa27c67e58e7d920028d3&amp;deviceId=XlAqcFSXarsDADPOZC3hu7M8&amp;platform=android&amp;v=5.0</td><td><ul><li>Device ID (likely generated by the app itself)</li><li>Platform: Android</li><li>Platform version: 9</li><li>MNC: Wifi</li></ul><p>Below are fields with interesting names, but contains no meaningful data:</p><ul><li>mnc (mobile network code, which could identify the mobile data service provider)</li><li>lng (longitude)</li><li>bssid (hardware serial number of the Wifi access point)</li><li>carrier</li><li>lat (latitude)</li></ul></td></tr></tbody></table><table headertable="true"><thead><tr><th colspan="2">Umeng</th></tr></thead><tbody><tr><td>Request endpoint</td><td>Information sent</td></tr><tr><td><a href="https://plbslog.umeng.com/umpx_internal"><u>https://plbslog.umeng.com/umpx_internal</u></a></td><td>Encrypted content</td></tr><tr><td>https://ulogs.umeng.com/unify_logs</td><td><ul><li>OS build ID</li><li>Device model</li><li>OS version</li></ul></td></tr><tr><td>http://alog.umeng.com/app_logs</td><td>Encrypted content</td></tr></tbody></table><table headertable="true"><thead><tr><th colspan="2">Xiaomi</th></tr></thead><tbody><tr><td>Request endpoint</td><td>Sensitive information sent</td></tr><tr><td>POST <a href="https://register.xmpush.global.xiaomi.com/pass/v2/register"><u>https://register.xmpush.global.xiaomi.com/pass/v2/register</u></a></td><td><ul><li>Hardware board model: universal7870</li><li>Device brand: samsung</li><li>Device model: SM-A320Y</li><li>OS version string: 9-eng.McFy.20190908.193312</li><li>(suspected) Google Analytics ID: 3575bc02-xxxx-xxxx-xxxx-xxxxxxxxxxxx</li><li>(suspected) Device ID</li></ul></td></tr><tr><td>GET <a href="http://resolver.msg.global.xiaomi.net/gslb/"><u>http://resolver.msg.global.xiaomi.net/gslb/</u></a></td><td>GET parameters:<ul><li>Network type: wifi</li><li>Country code: tw</li><li>OS version: 28</li><li>Device model and OS version string: SM-A320Y%3Aeng.McFy.20190908.193312</li></ul></td></tr></tbody></table>

Table 5: Third-party data collection in Douyin, including the data transmission endpoints, and sensitive information the transmission contained.

##### First-party tracking

Douyin sent device information to its own servers. Some fields were sent using simple HTTPS, others were sent encrypted using its proprietary “ttEncrypt” algorithm. Details on the algorithm can be found in the “API data encryption using native libraries_”_ section below.

Following is a list of device information fields sent using simple HTTPS. For brevity and ease of understanding, if a field appears in multiple requests, we only list it once.

<table headertable="true"><thead><tr><th>Request field name</th><th>Captured value</th><th>Remarks</th></tr></thead><tbody><tr><td>language</td><td>zh</td><td></td></tr><tr><td>os_api</td><td>28</td><td></td></tr><tr><td>os_version</td><td>9</td><td></td></tr><tr><td>resolution</td><td>720*1280</td><td></td></tr><tr><td>dpi</td><td>311</td><td></td></tr><tr><td>device_type</td><td>SM-A320Y</td><td></td></tr><tr><td>device_brand</td><td>samsung</td><td></td></tr><tr><td>device_platform</td><td>android</td><td></td></tr><tr><td>openudid</td><td>723364xxxxxxxxxx</td><td></td></tr><tr><td>abi</td><td>armeabi-v7a</td><td></td></tr><tr><td>cpu_model</td><td>samsungexynos7870</td><td></td></tr><tr><td>latitude</td><td>null</td><td></td></tr><tr><td>webcast_language</td><td>zh-TW</td><td></td></tr><tr><td>webcast_locale</td><td>zh_TW_%23Hant</td><td></td></tr><tr><td>longitude</td><td>null</td><td></td></tr><tr><td>volume</td><td>0.06666666666666667</td><td></td></tr><tr><td>is_cold_start</td><td>1</td><td></td></tr><tr><td>country</td><td>TW</td><td></td></tr><tr><td>rom</td><td>eng.McFy.20190908.193312</td><td></td></tr><tr><td>tz_name</td><td>America%2FToronto</td><td></td></tr><tr><td>device_id</td><td>707xxxxxxxx</td><td></td></tr><tr><td>region</td><td>tw</td><td></td></tr><tr><td>rom_version</td><td>28</td><td></td></tr><tr><td>req_ip</td><td>128.100.x.x</td><td>External IP address of the test phone</td></tr><tr><td>build_serial</td><td>5203b4xxxxxxxxxx</td><td></td></tr><tr><td>google_aid</td><td>3575bc02-xxxx-xxxx-xxxx-xxxxxxxxxxxx</td><td>Google advertising ID</td></tr></tbody></table>

Table 6: List of device information fields sent by Douyin using simple HTTPS.

List of device information fields sent using ttEncrypt on top of HTTPS:

<table headertable="true"><thead><tr><th>Request field name</th><th>Captured value</th></tr></thead><tbody><tr><td>appkey</td><td>57bfa27c67e58e7d920028d3</td></tr><tr><td>openudid</td><td>0</td></tr><tr><td>sdk_version</td><td>2.5.6.3</td></tr><tr><td>package</td><td>com.ss.android.ugc.aweme</td></tr><tr><td>channel</td><td>aweGW</td></tr><tr><td>display_name</td><td>抖音短视频</td></tr><tr><td>app_version</td><td>9.6.0</td></tr><tr><td>version_code</td><td>960</td></tr><tr><td>timezone</td><td>-5</td></tr><tr><td>access</td><td>wifi</td></tr><tr><td>os</td><td>Android</td></tr><tr><td>os_version</td><td>9</td></tr><tr><td>os_api</td><td>28</td></tr><tr><td>device_model</td><td>unknown</td></tr><tr><td>device_brand</td><td>unknown</td></tr><tr><td>device_manufacturer</td><td>unknown</td></tr><tr><td>language</td><td>zh</td></tr><tr><td>resolution</td><td>1280×720</td></tr><tr><td>display_density</td><td>mdpi</td></tr><tr><td>density_dpi</td><td>311</td></tr><tr><td>mc</td><td>24:xx:xx:xx:xx:xx</td></tr><tr><td>carrier</td><td>unknown</td></tr><tr><td>mcc_mnc</td><td>101</td></tr><tr><td>clientudid</td><td>17604b3e-xxxx-xxxx-xxxx-xxxxxxxxxxxx</td></tr><tr><td>sig_hash</td><td>aea615ab910015038f73c47e45d21466</td></tr><tr><td>aid</td><td>1128</td></tr><tr><td>push_sdk</td><td>[1, 2, 6, 7, 8, 9]</td></tr><tr><td>rom</td><td>eng.McFy.20190908.193312</td></tr><tr><td>release_build</td><td>7694899_20200115</td></tr><tr><td>update_version_code</td><td>9602</td></tr><tr><td>manifest_version_code</td><td>960</td></tr><tr><td>app_version_minor</td><td></td></tr><tr><td>cpu_abi</td><td>armeabi-v7a</td></tr><tr><td>build_serial</td><td></td></tr><tr><td>sim_serial_number</td><td>[]</td></tr><tr><td>not_request_sender</td><td>0</td></tr><tr><td>rom_version</td><td>unknown</td></tr><tr><td>region</td><td>TW</td></tr><tr><td>tz_name</td><td>America\/Toronto</td></tr><tr><td>tz_offset</td><td>-18000</td></tr><tr><td>sim_region</td><td>xx</td></tr><tr><td>google_aid</td><td>26716a75-xxxx-xxxx-xxxx-xxxxxxxxxxxx</td></tr></tbody></table>

Table 7: List of device information fields sent by Douyin using ttEncrypt on top of HTTPS.

Notably, using ttEncrypt, Douyin concealed the sending of device MAC address, MCC, and MNC. The MAC address could be used to uniquely identify a device. MCC (Mobile Country Code) and MNC (Mobile Network Code) are used to uniquely identify a mobile network operator (carrier).

In addition to device information, Douyin transmitted more “event” data using ttEncrypt when compared to TikTok. These “events” seemed to represent various actions taken by the app during execution, for instance:

*   “SDK_launch” event seemed to denote launch of a “SDK” app component, which recorded some relevant parameters and the precise timestamp.
*   “location_status” event checked if the device’s GPS was enabled and returned the result.

However, even though Douyin collected a lot of device information, it did not seem to upload contacts, read files on the phone, track physical location, or listen to the microphone in the background.

There were also some Interesting nuances to the data fields:

*   In many requests, some data fields (but not all) were repeated in the same request.
*   There were physical location fields but only null values were sent. Douyin requests location permission on startup. Even if we grant it location permission at this point, future requests to the same endpoint still send null values.

#### TikTok

From intercepted network traffic[6](#fn6), we can see Musically contacts the following third-party tracker services:

<table headertable="true"><thead><tr><th>Tracker service</th><th>Intercepted network request URL</th></tr></thead><tbody><tr><td>Facebook</td><td>https://graph.facebook.com/v4.0/597615686992125/activities</td></tr><tr><td>AppsFlyer</td><td><a href="https://t.appsflyer.com/api/v4/androidevent?buildnumber=4.8.20&amp;app_id=com.zhiliaoapp.musically"><u>https://t.appsflyer.com/api/v4/androidevent?buildnumber=4.8.20&amp;app_id=com.zhiliaoapp.musically</u></a><p>https://api.appsflyer.com/install_data/v3/com.zhiliaoapp.musically?devkey=XY8Lpakui8g4kBcposRgxA&amp;device_id=1581454754910-1790280114805923294</p></td></tr><tr><td>Google Analytics for Firebase</td><td>https://app-measurement.com/a</td></tr></tbody></table>

Table 8: Third-party trackers used by TikTok.

We will examine the information sent to each tracker service in the following subsections.

##### Facebook Mobile SDK

The Facebook Mobile SDK (Software Development Kit) is often used in apps. [Previous work](https://http.icsi.berkeley.edu/icsi/node/5893) has found that the Facebook Mobile SDK is embedded by 31% of the 14,599 apps researchers have analyzed.

We found Musically embeds the Facebook Mobile SDK and uses its [App Events](https://developers.facebook.com/docs/app-events/getting-started-app-events-android) feature to send usage information to Facebook. According to its official documentation, “Facebook App Events allows your app or web page to track events, such as a person installing your app or completing a purchase.”

Also according to its documentation, there are three types of App Events: “Automatically Logged Events,” “Standard Events,” and “Custom Events.” Automatically Logged Events are sent by the SDK automatically and therefore uniform across different apps. The other two types are more interesting to us because it is up to the developers to decide what information to send and when to send them.

When scrolling through TikTok’s feed, we intercepted multiple requests to the endpoint [https://graph.facebook.com/v4.0/597615686992125/activities](https://graph.facebook.com/v4.0/597615686992125/activities) with content fields listed below (note the examples below are a selection of fields we found to be of interest.)

<table headertable="true"><thead><tr><th>Field</th><th>Sample value</th><th>Remark</th></tr></thead><tbody><tr><td>fb_content_id</td><td>6856850517394492677</td><td>According to the <a href="https://developers.facebook.com/docs/marketing-api/app-event-api/v2.6"><u>documentation</u></a>, this field represents the developer-defined content identifier.<p>For TikTok, the value represents the video ID that the user is viewing. We know this because this particular request was sent exactly at the time when we were viewing this video on the test phone <a href="https://www.tiktok.com/@beardyzig/video/6856850517394492677"><u>https://www.tiktok.com/@beardyzig/video/6856850517394492677</u></a></p><p>In our observation, this kind of event is not sent for <em>all</em> video views, but only <em>most</em> video views.</p><p><strong>The implication of this information being sent is: Facebook actually knows what videos are viewed by the user</strong>. If the user later logs into a TikTok account linked to a Facebook account, Facebook is able to correlate the viewing history with a particular Facebook user.</p><p>However, this kind of cross-platform tracking is not new for the advertisement and tracking industry. For example, Facebook has been <a href="https://www.cnbc.com/2018/03/19/how-facebook-ad-tracking-and-targeting-works.html"><u>tracking webpage views using Facebook widgets embedded in non-Facebook webpages</u></a> for a long time.</p></td></tr><tr><td>_eventName</td><td>fb_mobile_content_view</td><td>This field represents the type of event being sent. “fb_mobile_content_view” is one of the standard types that Facebook already defines in their <a href="https://developers.facebook.com/docs/marketing-api/app-event-api/?locale=en_US#standard-event-names"><u>documentation</u></a>.</td></tr><tr><td>installer_package</td><td>com.aurora.store</td><td>The source package that installed this app.</td></tr><tr><td>extinfo</td><td>[“a2”, “com.zhiliaoapp.musically”,2021407050,”14.7.5″,”9″,”SM-A320Y”,”zh_TW”,”EDT”,””,720,1280,”1.94″,8,10,4, “America\/Toronto”]</td><td>Documented here: <a href="https://developers.facebook.com/docs/graph-api/reference/application/activities/#parameters"><u>https://developers.facebook.com/docs/graph-api/reference/application/activities/#parameters</u></a><p>In sequence, these values represent:</p><ul><li>extinfo version (a2 for android)</li><li>app package name</li><li>short version (int or string)</li><li>long version</li><li>OS version</li><li>device model name</li><li>locale</li><li>timezone abbreviation</li><li>carrier</li><li>screen width</li><li>screen height</li><li>screen density (float decimal)</li><li>number of CPU cores</li><li>external storage size in GB</li><li>free space on external storage in GB</li><li>device timezone</li></ul></td></tr></tbody></table>

Table 9: Information sent to Facebook by TikTok.

Outside of these standard event types, developers can also define their own using Custom Events. We searched the source code for “fb_mobile_content_view”, and were able to find other types of events logged to Facebook in the same Java class, which includes both Standard Events and Custom Events. From the network traffic analysis, we did not observe every kind of Custom Events defined in the source code. The actual timing of sending these events are usually a few seconds after the actions are performed.

<table headertable="true"><thead><tr><th>Custom Event name<br>* marked are events that we actually observe when using the app, otherwise they only exist in source code.</th><th>Occasion</th></tr></thead><tbody><tr><td>fb_mobile_add_to_wishlist</td><td></td></tr><tr><td>fb_mobile_rate</td><td></td></tr><tr><td>Login</td><td></td></tr><tr><td>fb_mobile_add_to_cart*</td><td>Liking a post</td></tr><tr><td>Subscribe*</td><td>Following a user</td></tr><tr><td>fb_mobile_purchase</td><td></td></tr><tr><td>fb_mobile_content_view*</td><td>Viewing a post</td></tr><tr><td>fb_mobile_achievement_unlocked</td><td></td></tr><tr><td>2d_rr_user</td><td>User uses the app both today and yesterday</td></tr><tr><td>F1</td><td rowspan="4">Related to the video view count of the user over many days.</td></tr><tr><td>F10*</td></tr><tr><td>F30</td></tr><tr><td>F50</td></tr><tr><td>fb_mobile_initiated_checkout</td><td></td></tr><tr><td>fb_mobile_add_payment_info</td><td></td></tr></tbody></table>

Table 10: “Custom Events” sent to Facebook by TikTok.

For Automatically Logged Events, we did not find them to contain sensitive information.

##### Firebase Analytics

Similar to Facebook Mobile SDK, Firebase Analytics also tracks users by [automatically collected events](https://support.google.com/firebase/answer/9234069?hl=en&ref_topic=6317484) (like in-app purchases) and custom events (defined by the developer). Firebase Analytics communicates with the endpoint [https://app-measurement.com/a](https://app-measurement.com/a) . Automatically collected events contain fields

HTTP requests towards Firebase Analytics are [heavily compressed](https://stackoverflow.com/questions/54461349/how-to-decrypt-firebase-requests-to-app-measurement-com/54463682#54463682), so we could not work out a complete list of events and data items sent to Firebase Analytics. However, we were able to confirm a part of them from the uncompressed parts of the request and source code.

There were four requests sent towards Firebase Analytics between first-time launching the app and reaching the main content feed. The first two of them only seemed to contain a subset of all “predefined user dimensions” specified in [Firebase’s documentation](https://support.google.com/firebase/answer/9268042?hl=en&ref_topic=6317484). Other parts of the first two requests were undecipherable to us, presumably due to the compression.

The third request contained two fields that we could decipher: “file_dirs_num” and “did”. Source code analysis suggests that “field_dirs_num” seems to represent the number of external storages present on the phone, while “did” is some kind of device ID.

[![](https://citizenlab.ca/wp-content/uploads/2021/03/img17.png)](https://citizenlab.ca/wp-content/uploads/2021/03/img17.png)

Figure 17: Sources of “file_dirs_num” and “did” in Musically.

The fourth request contained the string “Video_content”, and a field “item_id” which specified “6908178681542216965”:

[![](https://citizenlab.ca/wp-content/uploads/2021/03/img18.png)](https://citizenlab.ca/wp-content/uploads/2021/03/img18.png)

Figure 18: Musically logs the ID of the first video viewed by the user to Firebase Analytics.

The numeric string “6908178681542216965” corresponded to the [ID of the video](https://www.tiktok.com/@laurierlachance/video/6908178681542216965) first shown in our content feed.

By looking for other calls to the function m134968a which sent “Video_content”, we found other “events” sent to Firebase Analytics:

[![](https://citizenlab.ca/wp-content/uploads/2021/03/img19.png)](https://citizenlab.ca/wp-content/uploads/2021/03/img19.png)

Figure 19: By looking for other calls to the function “m134968a” which sent “Video_content”, we found other “events” sent to Firebase Analytics.

In the following table, we list all other events found by the method described above, and the occasion these events were sent if possible.

<table headertable="true"><thead><tr><th>Event name</th><th>Occasion</th></tr></thead><tbody><tr><td>login</td><td></td></tr><tr><td>like</td><td></td></tr><tr><td>follow</td><td></td></tr><tr><td>UPV</td><td></td></tr><tr><td>Video_content</td><td>User views a video</td></tr><tr><td>d2_rr_user</td><td>User uses the app both today and yesterday</td></tr><tr><td>F1</td><td rowspan="4">Related to the video view count of the user over many days.</td></tr><tr><td>F10</td></tr><tr><td>F30</td></tr><tr><td>F50</td></tr><tr><td>search</td><td>Same as fb_mobile_search. This event additionally includes the search term.</td></tr></tbody></table>

Table 11: Musically’s custom Firebase Analytics events.

However, we did not observe any of those custom events other than “Video_content” being sent, even when we tried to perform some of the actions specified by the name of the event, such as “login”. The “Video_content” event was also only ever observed once, when the app first launches. Thus, in summary, even though TikTok tried to log similar events to Firebase Analytics as it log to Facebook, only a small subset of them were actually logged.

##### AppsFlyer

In our capture, AppsFlyer’s traffic pattern was more erratic. One request was always sent whenever the user launches the app, and a few other requests were sent when we used the app. However, there did not seem to be any user action that can reliably trigger a request to AppsFlyer. Here we document the sensitive information sent in app startup:

<table headertable="true"><thead><tr><th>Field name</th><th>Captured value</th><th>Remarks</th></tr></thead><tbody><tr><td>country</td><td>TW</td><td></td></tr><tr><td>isFirstCall</td><td>true</td><td></td></tr><tr><td>timepassedsincelastlaunch</td><td>-1</td><td></td></tr><tr><td>operator</td><td>(empty string)</td><td></td></tr><tr><td>network</td><td>WIFI</td><td></td></tr><tr><td>open_referrer</td><td>android-app:\/\/com.android.launcher3</td><td>The application that opened this application, in this case it contains the App ID of LineageOS’s default home screen application launcher.</td></tr><tr><td>uid</td><td>1597419074570-xxxxxxxxxxxxxx</td><td></td></tr><tr><td>isGaidWithGps</td><td>true</td><td></td></tr><tr><td>lang_code</td><td>zh</td><td></td></tr><tr><td>installDate</td><td>2020-01-25_144116+0000</td><td>The time when the app is first installed.</td></tr><tr><td>firstLaunchDate</td><td>2020-08-14_153116+0000</td><td>The time when the app is first launched.</td></tr><tr><td>model</td><td>SM-A320Y</td><td>Device model name</td></tr><tr><td>lang</td><td>中文</td><td>Translates to “Chinese.”</td></tr><tr><td>brand</td><td>samsung</td><td></td></tr><tr><td>batteryLevel</td><td>100.0</td><td></td></tr><tr><td>deviceType</td><td>userdebug</td><td></td></tr><tr><td>product</td><td>a3y17lte</td><td>Device model codename</td></tr><tr><td>sensors</td><td><code>[{<br>"sT": 4<br>"sVS": [-0.112399206, -0.03604105, 0.055588737],<br>"sN": "K6DS3TR Gyroscope",<br>"sV": "STM"<br>"sVE": [0.004276057, 0.05375614, 0.08674286]<br>}, {<br>"sT": 1<br>"sVS": [2.4349031, 2.1859062, 8.935161],<br>"sN": "K6DS3TR Accelerometer",<br>"sV": "STM",<br>"sVE": [2.0973208, 2.1404164, 9.655815]<br>}, {<br>"sT": 2,<br>"sVS": [26.64, -66, 24.539999],<br>"sN": "AK09916C Magnetometer",<br>"sV": "Asahi Kasei Microdevices"<br>"sVE": [26.76, -64.74, 25.38]<br>}]</code></td><td>This field includes the manufacturers and model names of gyroscope, accelerometer and magnetometer on the device. Those information may be used to help identify a particular phone model, as different phone models embed different models of sensors.<p>Sensors data does not require the user granting any permission on Android, they are granted by the Android system upon installation.</p><p>The numeric values shown on the left may be actual data from the sensors, because these sets of three values correspond to the three spatial axes of data that the sensors are sensing, however, we are not certain.</p><p><a href="https://www.ccs.neu.edu/home/noubir/publications-local/NVBN2016.pdf"><u>Previous work</u></a> found that, if these sensor data are collected continuously from a device traveling on a vehicle, one is able to to determine the route on which it is traveling to a highly accurate degree, without needing additional location information.</p></td></tr><tr><td>cpu_abi</td><td>armeabi-v7a</td><td></td></tr><tr><td>build_display_id</td><td>lineage_a3y17lte-userdebug 9 PQ3A.190801.002 eng.McFy.20190908.193312 test-keys</td><td></td></tr><tr><td>dim</td><td><code>{<br>"size": "2",<br>"ydp": "309.638",<br>"xdp": "309.966"<br>"y_px": "1280",<br>"x_px": "720",<br>"d_dpi": "311"<br>}</code>,</td><td>Screen dimensions</td></tr><tr><td>installer_package</td><td>com.aurora.store</td><td>Source package that installed this app</td></tr><tr><td>android_id</td><td>6cxxxxxxxxxxxxxx</td><td></td></tr><tr><td>device</td><td>a3y17lte</td><td>Device model codename</td></tr></tbody></table>

Table 12: Sensitive information being sent to AppsFlyer on Musically’s startup.

##### Musically first-party tracking

Musically also sends device information to their own servers. Same as Douyin, some fields were sent using simple HTTPS, others were sent encrypted using its proprietary “ttEncrypt” algorithm on top of existing HTTPS. Details on the algorithm can be found in the “API data encryption using native libraries” section below.

List of device information fields sent using simple HTTPS:

<table headertable="true"><thead><tr><th>Request field name</th><th>Captured value</th></tr></thead><tbody><tr><td>ad_area</td><td>720×1233</td></tr><tr><td>os_api</td><td>28</td></tr><tr><td>device_platform</td><td>android</td></tr><tr><td>os_version</td><td>9</td></tr><tr><td>display_density</td><td>720×1280</td></tr><tr><td>dpi</td><td>311</td></tr><tr><td>device_brand</td><td>samsung</td></tr><tr><td>device_type</td><td>SM-A320Y</td></tr><tr><td>display_dpi</td><td>311</td></tr><tr><td>density</td><td>1.94375</td></tr><tr><td>ac</td><td>wifi</td></tr><tr><td>channel</td><td>googleplay</td></tr><tr><td>language</td><td>zh</td></tr><tr><td>iid</td><td>6786xxxxxxxxxxxxxxx</td></tr><tr><td>device_id</td><td>6786xxxxxxxxxxxxxxx</td></tr><tr><td>openudid</td><td>6c10xxxxxxxxxxxx</td></tr><tr><td>gaid</td><td>3575bc02-xxxx-xxxx-xxxx-xxxxxxxxxxxx</td></tr><tr><td>android_id</td><td>6c10xxxxxxxxxxxx</td></tr><tr><td>ad_user_agent</td><td>Mozilla%2F5.0+%28Linux%3B+Android+9%3B+SM-A320Y+Build%2FPQ3A.190801.002%3B+wv%29+AppleWebKit%2F537.36+%28KHTML%2C+like+Gecko%29+Version%2F4.0+Chrome%2F76.0.3809.111+Mobile+Safari%2F537.36</td></tr><tr><td>app_language</td><td>zh-Hant</td></tr><tr><td>locale</td><td>zh-Hant-TW</td></tr><tr><td>resolution</td><td>720*1280</td></tr><tr><td>ac2</td><td>wifi</td></tr><tr><td>sys_region</td><td>TW</td></tr><tr><td>is_my_cn</td><td>0</td></tr><tr><td>timezone_name</td><td>America%2FToronto</td></tr><tr><td>timezone_offset</td><td>-18000</td></tr><tr><td>op_region</td><td>TW</td></tr><tr><td>install_id</td><td>6786xxxxxxxxxxxxxxx</td></tr><tr><td>google_aid</td><td>3575bc02-xxxx-xxxx-xxxx-xxxxxxxxxxxx</td></tr><tr><td>timezone</td><td>-5</td></tr></tbody></table>

Table 13: List of device information fields sent using simple HTTPS by TikTok.

List of device information fields sent using ttEncrypt on top of HTTPS:

<table headertable="true"><thead><tr><th>Request field name</th><th>Captured value</th></tr></thead><tbody><tr><td>appkey</td><td>5559e28267e58eb4c1000012</td></tr><tr><td>openudid</td><td>d97cxxxxxxxxxxxx</td></tr><tr><td>sdk_version</td><td>2.5.5.8</td></tr><tr><td>package</td><td>com.zhiliaoapp.musically</td></tr><tr><td>channel</td><td>googleplay</td></tr><tr><td>display_name</td><td>TikTok</td></tr><tr><td>app_version</td><td>14.7.5</td></tr><tr><td>version_code</td><td>2021407050</td></tr><tr><td>timezone</td><td>-5</td></tr><tr><td>access</td><td>wifi</td></tr><tr><td>os</td><td>Android</td></tr><tr><td>os_version</td><td>9</td></tr><tr><td>os_api</td><td>28</td></tr><tr><td>device_model</td><td>unknown</td></tr><tr><td>device_brand</td><td>unknown</td></tr><tr><td>language</td><td>zh</td></tr><tr><td>resolution</td><td>1280×720</td></tr><tr><td>display_density</td><td>mdpi</td></tr><tr><td>density_dpi</td><td>311</td></tr><tr><td>mc</td><td>02:00:00:00:00:00</td></tr><tr><td>carrier</td><td>unknown</td></tr><tr><td>mcc_mnc</td><td>101</td></tr><tr><td>clientudid</td><td>2563608d-xxxx-xxxx-xxxx-xxxxxxxxxxxx</td></tr><tr><td>sig_hash</td><td>194326e82c84a639a52e5c023116f12a</td></tr><tr><td>aid</td><td>1233</td></tr><tr><td>push_sdk</td><td>[1, 2, 6, 7, 8, 9]</td></tr><tr><td>rom</td><td>eng.McFy.20190908.193312</td></tr><tr><td>release_build</td><td>688b613_20200121</td></tr><tr><td>update_version_code</td><td>2021407050</td></tr><tr><td>manifest_version_code</td><td>2021407050</td></tr><tr><td>cpu_abi</td><td>armeabi-v7a</td></tr><tr><td>sim_serial_number</td><td>[]</td></tr><tr><td>not_request_sender</td><td>0</td></tr><tr><td>rom_version</td><td>unknown</td></tr><tr><td>region</td><td>TW</td></tr><tr><td>tz_name</td><td>America\/Toronto</td></tr><tr><td>tz_offset</td><td>-18000</td></tr><tr><td>sim_region</td><td>xx</td></tr><tr><td>google_aid</td><td>26716a75-xxxx-xxxx-xxxx-xxxxxxxxxx</td></tr><tr><td>app_language</td><td>zh-Hant</td></tr><tr><td>sys_region</td><td>TW</td></tr><tr><td>carrier_region</td><td>XX</td></tr><tr><td>timezone_name</td><td>東部標準時間</td></tr><tr><td>timezone_offset</td><td>-18000</td></tr></tbody></table>

Table 14: List of device information fields sent by TikTok using ttEncrypt on top of HTTPS.

### Comparison

When focusing purely on device information fields, Douyin itself (first-party) collects slightly more types of data than Musically:

*   cpu_model
*   volume
*   req_ip
*   MAC address
*   MCC and MNC

When we look at the bigger picture of all the data transmitted to their servers, Douyin sends more “events” data compared to TikTok. This information may allow Douyin to more precisely infer the user’s in-app behavior.

Douyin and Musically also used different advertisement and tracking services, choosing ones that are common in their operating regions.

Our analysis of security issues in TikTok and Douyin resulted in the following high-level findings:

*   All of TikTok and most of Douyin’s network traffic were adequately protected using HTTPS. For some data, an additional layer of encryption, which we dubbed “ttEncrypt,” was employed. We engineered a way to intercept the clear-text data before they were encrypted with ttEncrypt. We examined the data and found no clear reason why these data had to be encrypted again on top of the existing transport encryption provided by HTTPS, as these ttEncrypt-ed data were not confidential.
*   Most of TikTok and Douyin’s API requests are protected with a custom signature in the HTTP header named “X-Gorgon.” The signature is generated using a native library module, which made it difficult for us to understand its inner workings. We think the purpose of this signature is to prevent third-party programs from imitating and sending TikTok/Douyin API requests.
*   We found that there are people selling and buying third-party implementations of ttEncrypt and X-Gorgon algorithms. These third-party implementations might be produced to serve the need of bots (programs disguised as real users).
*   Douyin loads some of its code components via the Internet. It can also update itself without any user interaction. Such code loading was adequately protected using HTTPS, making it difficult for attackers to inject malicious code in the loading process. However, this feature is a security issue because it bypasses the operating system’s and user’s control of what code could run on the device. TikTok does not include this feature.
*   Overall, TikTok includes some unusual internal designs, but does not otherwise exhibit overtly malicious behavior. Douyin’s dynamic code loading feature can be seen as malicious, as it bypasses the system installation process, but this feature is also commonly seen in Chinese apps and generally accepted in the Chinese market.

In the subsections below we document security designs and properties of interest that we found in the apps.

HTTPS
-----

We found that all of TikTok’s traffic was encrypted using HTTPS. Douyin also encrypts most of its traffic using HTTPS, but leaves some analytics data and data from Content Delivery Networks unencrypted. In the following table, we list the unencrypted network requests and the information they contain.

<table headertable="true"><thead><tr><th>Purpose</th><th>URL</th><th>Sensitive information included</th></tr></thead><tbody><tr><td>Analytics</td><td><a href="http://alog.umeng.com/app_logs"><u>http://alog.umeng.com/app_logs</u></a></td><td>System version and device model</td></tr><tr><td>Analytics</td><td><a href="http://amdcopen.m.taobao.com/amdc/mobileDispatch?appkey=umeng%3A57bfa27c67e58e7d920028d3&amp;deviceId=X9oLqPe1hRUDAG7Iab8fPK1k&amp;platform=android&amp;v=5.0"><u>http://amdcopen.m.taobao.com/amdc/mobileDispatch</u></a></td><td><ul><li>System version and device model</li><li>This request also included these sensitive fields, but with empty values, indicating that under some condition these information might also be transmitted:<ul><li>MNC</li><li>Longitude and latitude</li><li>BSSID</li><li>Mobile carrier name</li></ul></li></ul></td></tr><tr><td>Analytics</td><td><a href="http://adash.man.aliyuncs.com/man/api"><u>http://adash.man.aliyuncs.com/man/api</u></a></td><td>System version and device model</td></tr><tr><td>Media file on CDN</td><td>http://sf3-webcast-dycdn.byteimg.com/*</td><td>None</td></tr><tr><td>Media file on CDN</td><td>http://sf6-webcast-dycdn.byteimg.com/*</td><td>None</td></tr></tbody></table>

Table 15: List of unencrypted network requests sent by Douyin.

**Server Locations**
--------------------

Below is a list of servers that TikTok connected to during startup and user registration:

<table headertable="true"><thead><tr><th>Hostname</th><th>Remarks</th></tr></thead><tbody><tr><td><a href="http://graph.facebook.com/"><u>graph.facebook.com</u></a></td><td>Facebook Graph API</td></tr><tr><td><a href="http://api2-16-h2.musical.ly/"><u>api2-16-h2.musical.ly</u></a></td><td>TikTok’s API server</td></tr><tr><td><a href="http://log2.musical.ly/"><u>log2.musical.ly</u></a></td><td>TikTok’s log server</td></tr><tr><td><a href="http://android.clients.google.com/"><u>android.clients.google.com</u></a></td><td>Setting up Google’s push notification system</td></tr><tr><td><a href="http://t.appsflyer.com/"><u>t.appsflyer.com</u></a></td><td>AppsFlyer analytics</td></tr><tr><td><a href="http://api.appsflyer.com/"><u>api.appsflyer.com</u></a></td><td>AppsFlyer analytics</td></tr><tr><td><a href="http://xlog-va.musical.ly/"><u>xlog-va.musical.ly</u></a></td><td>TikTok’s log server</td></tr><tr><td><a href="http://dm16.musical.ly/"><u>dm16.musical.ly</u></a></td><td>TikTok’s server</td></tr><tr><td><a href="http://sdfp-va.musical.ly/"><u>sdfp-va.musical.ly</u></a></td><td>TikTok’s server</td></tr><tr><td><a href="http://app-measurement.com/"><u>app-measurement.com</u></a></td><td>Google Analytics for Firebase</td></tr><tr><td><a href="http://api2.musical.ly/"><u>api2.musical.ly</u></a></td><td>TikTok’s API server</td></tr><tr><td><a href="http://webcast.musical.ly/"><u>webcast.musical.ly</u></a></td><td>TikTok’s server</td></tr><tr><td><a href="http://dm-maliva16.byteoversea.com/"><u>dm-maliva16.byteoversea.com</u></a></td><td>TikTok’s server</td></tr><tr><td><a href="http://applog.musical.ly/"><u>applog.musical.ly</u></a></td><td>TikTok’s log server</td></tr><tr><td><a href="http://dm16.byteoversea.com/"><u>dm16.byteoversea.com</u></a></td><td>TikTok’s server</td></tr><tr><td><a href="http://p16.muscdn.com/"><u>p16.muscdn.com</u></a></td><td>TikTok’s CDN server</td></tr><tr><td><a href="http://gecko-va.musical.ly/"><u>gecko-va.musical.ly</u></a></td><td>TikTok’s CDN server</td></tr><tr><td><a href="http://mon.musical.ly/"><u>mon.musical.ly</u></a></td><td>TikTok’s log server</td></tr><tr><td><a href="http://abtest-va-tiktok.byteoversea.com/"><u>abtest-va-tiktok.byteoversea.com</u></a></td><td>TikTok’s server</td></tr><tr><td><a href="http://verification-va.musical.ly/"><u>verification-va.musical.ly</u></a></td><td>TikTok’s server</td></tr><tr><td><a href="http://sf16-security-va.ibytedtos.com/"><u>sf16-security-va.ibytedtos.com</u></a></td><td>TikTok’s server</td></tr><tr><td><a href="http://imapi-16.musical.ly/"><u>imapi-16.musical.ly</u></a></td><td>TikTok’s server</td></tr></tbody></table>

Table 16: A list of servers that TikTok connected to during startup and user registration.

All of TikTok’s server domain names are resolved to Akamai Content Delivery Network (CDN) “CNAME”s, which are another set of domain names. CDN services usually sit between the users and the actual service providers to offer caching and load balancing capabilities. This however also means that we would not be able to know the real IP address of TikTok’s servers. Without the real IP addresses, there is no way for us to deduce the server’s physical locations. CDN services also usually direct user connections towards servers located near the users. In our case, all of our connections were directed towards servers in North America.

TikTok API Security
-------------------

Almost all of TikTok’s HTTP API requests contained a “X-Gorgon” header. Its value looked something like this:

`X-Gorgon: 040140f50001e502b3eef3a93b9af32139323006cd23e9b4ac3e`

The values were different across different requests, but the length remained the same. If we omitted the header, the server would not process the request. From those characteristics, we suspect this header likely contains custom digital signatures.

From its source code, one of the steps in generating the X-Gorgon value involves calling the `com.p420ss.sys.ces.C45485a.leviathan()` function:

[![](https://citizenlab.ca/wp-content/uploads/2021/03/img20.png)](https://citizenlab.ca/wp-content/uploads/2021/03/img20.png)

Figure 20: TikTok source code calls the function “leviathan” as one of the steps in generating X-Gorgon value.

We quickly see that leviathan() is implemented in a native library module “libcms.so”:

[![](https://citizenlab.ca/wp-content/uploads/2021/03/img21.png)](https://citizenlab.ca/wp-content/uploads/2021/03/img21.png)

Figure 21: The leviathan() function is implemented in a native library module “libcms.so”.

Compared to most of the application code, which are implemented in Java, [native library modules](https://developer.android.com/ndk) are harder to reverse engineer. Given this characteristic, we think the purpose of the signature is to make sure the HTTP API request originates from a legitimate installation of the app on a real phone, instead of automated programs which could be massively duplicated to run a bot network.

This signature protection is used in both TikTok and Douyin. The libcms.so file[7](#fn7) contained in TikTok and Douyin are also identical (with MD5 hash 2b832021d421a6d63db0e4075d250fa3), which indicates that X-Gorgon signature is generated in the same way for both TikTok and Douyin.

Such a protection mechanism shows the developer’s effort in weeding out automated requests (like bots) and securing their APIs against reverse engineering or hacking attempts. However, it is not a perfect solution to these problems, as we show a way to send automated search requests in the “Search Censorship” section below. With more effort, a persistent attacker may also be able to completely reverse engineer the signature algorithm.

### Proprietary “ttEncrypt” API data encryption using native libraries

In the HTTP requests that we captured, most of them could be easily decoded (such as using [gzip](https://en.wikipedia.org/wiki/Gzip) compression algorithm). However, some of the requests were encoded using a special algorithm that we could not easily decode. For those requests, the “Content-Type” HTTP header, [which suggests the encoding used by the request body](https://www.w3.org/Protocols/rfc1341/4_Content-Type.html), were set to “application/octet-stream;tt-data=a”. “tt-data” is not a standardized encoding, indicating that it is likely proprietary to TikTok. This encoding may be used to conceal sending sensitive user or device information, so we tried to decode its actual content. In the rest of the section we describe how we decoded “tt-data” in our captured HTTP requests.

We searched TikTok’s code for “tt-data”. In one instance, it was used in ”sendEncryptLog” function:

[![](https://citizenlab.ca/wp-content/uploads/2021/03/img22.png)](https://citizenlab.ca/wp-content/uploads/2021/03/img22.png)

Figure 22: TikTok’s sendEnctyptLog function which calls the TTEncryptUtils module.

“sendEncryptLog” calls “com.bytedance.frameworks.core.encrypt.TTEncryptUtils.m26202a”, and if m26202a returned a empty value, log the error message “encryption failed”. From this behavior, we could infer that m26202a is the function that performs the encryption. We then examined the Java class TTEncryptUtils, in it, we could see m26202a further calls the “handleData” function, and that handleData function was declared to be a native function (implemented with a native library module):

[![](https://citizenlab.ca/wp-content/uploads/2021/03/img23.png)](https://citizenlab.ca/wp-content/uploads/2021/03/img23.png)

Figure 23: A critical part of the ttEncrypt algorithm is implemented in the native library module “libttEncrypt.so”.

The TTEncryptUtils class also includes a call to load native module “ttEncrypt.” We decided to refer to this encryption as “ttEncrypt” because of the module name.

Implementing functions using [native libraries](https://developer.android.com/ndk) prevents it from being reverse engineered using pure Java-based decompilation (which we used).[8](#fn8) It is a common technique used by developers to hide the function implementation, access low level system APIs, or improve performance.

To see what content was being encrypted, we used a Frida-based tool “[jnitrace](https://github.com/chame1eon/jnitrace)” to intercept data passed to and from the native library module, with the following command:

`jnitrace -l libttEncrypt.so -m spawn com.zhiliaoapp.musically -o out1.json`

This command launched the Musically app, we then operated the app normally. During our operation, if libttEncrypt.so (ttEncrypt’s module file) was invoked, jnitrace will print the invoking function and parameters on our console.

From jnitrace’s [output](https://git.citlab.utoronto.ca/china-censorship/project-mgmt/snippets/131), we observed ttEncrypt invoking memory allocation functions of [JNIEnv](https://docs.oracle.com/javase/7/docs/technotes/guides/jni/spec/design.html), which is an interface that wraps around the native module. JNIEnv facilitates data passage between the native module and the Java part of the application. The following was the sequence of JNIEnv functions called:

1.  `JNIEnv->GetByteArrayRegion`
2.  `JNIEnv->NewByteArray`
3.  `JNIEnv->SetByteArrayRegion`

The sequence corresponds to the following high level operations which we interpreted:

1.  `TTEncryptUtils.handleData()` was called, the caller passed in a pointer to the byte array containing the data to be encrypted.
2.  ttEncrypt allocated a new byte array to store the encrypted data.
3.  ttEncrypt filled the newly allocated byte array with encrypted data. From jnitrace’s output, we can see the data began with `74 63 02 01 3d d8 bf ec` , ended with `10 1a 9f ce 5c 25 f5 7b`, and had length 2148.

The encrypted data content in operation 3 was identical to the data we captured in a HTTP POST request towards https://log2.musical.ly/service/2/app_log/ , as shown in the screenshot below.

[![](https://citizenlab.ca/wp-content/uploads/2021/03/img24.png)](https://citizenlab.ca/wp-content/uploads/2021/03/img24.png)

Figure 24: We found the encrypted data being sent in a HTTPS request.

Working our way up, we can find the original unencrypted data in the byte array passed in operation 1. This data was gzip-compressed, as the sendEncryptLog function source code suggested. We decompressed it and obtained the following [content](https://github.com/citizenlab/tiktok-report-data/blob/main/ttencrypt-ed.json) (excerpt):

[![](https://citizenlab.ca/wp-content/uploads/2021/03/img25.png)](https://citizenlab.ca/wp-content/uploads/2021/03/img25.png)

Figure 25: Cleartext data that was later encrypted with ttEncrypt.

The data contained some device information fields unseen in other requests which were not using ttEncrypt, such as “rom”and “rom_version.” Using the same jnitrace-based approach, we unencrypted another [request](https://git.citlab.utoronto.ca/china-censorship/project-mgmt/-/snippets/135) towards https://log2.musical.ly/service/2/device_register/ . It duplicated all the fields found in the previous request towards [https://log2.musical.ly/service/2/app_log/](https://log2.musical.ly/service/2/app_log/) .

Sometimes ttEncrypt was also used to encrypt plain-text strings instead of gzip-compressed data. In our capture, those plain-text strings were:

*   `device_type=unknown&device_brand=unknown&uuid=306359525468212&openudid=d97c76f085e4c667`
*   `device_type=unknown`

All device information fields encrypted with ttEncrypt are documented in the “Musically first party data collection” section above.

Other than device information, other information encrypted using ttEncrypt includes

*   `"session_id"`
*   date and time
*   `"tag": "first_launch_time"`
*   `"label": "perf_monitor"`
*   `"event": "absdk_result"`

Given the names of these fields, this information seems to be used for performance and A/B testing parameters tracking.

We are uncertain why all this information was encrypted again on top of existing HTTPS, as they do not seem to be confidential. One motivation behind such design may be to make reverse engineering more difficult.

### “ttEncrypt” encryption in Douyin

We also captured similar “tt-data=a” HTTP requests when using Douyin. Using the same jnitrace-based approach, we captured and analyzed the unencrypted data. Compared to Musically, Douyin put more information into ttEncrypt-ed data. These data fields are documented in the “Douyin first-party data collection” section above.

We found the native module files libttEncrypt.so contained in Musically and Douyin were identical, both having the same MD5 checksum 30ab1a28e467d7d1c859c4e31aba8dc0. This indicates that the ttEncrypt encryption works in the same way across Musically and Douyin.

### Third-party implementations for hire

While researching X-Gorgon and ttEncrypt API protections, we discovered that there seems to be a market for third party implementations of these algorithms. These implementations claim to generate valid X-Gorgon or ttEncrypt values for TikTok and Douyin API requests. For instance, we came across [a code repository on GitHub](https://github.com/ch4kn/tiktok-device-register), which only included a text file pointing to a website and claimed that it can create 2,000 accounts a day.

There are many other [similar repositories](https://github.com/xsuspectx/tiktok-x-gorgon) on GitHub, which usually include descriptions claiming their ability to generate those values and contact details for purchase. From their descriptions, most of them seem to target Douyin. Accompanying these code repositories, we also found numerous Chinese technical [articles](https://blog.csdn.net/weixin_41257505/article/details/106527835) on Douyin bot development.

Descriptions in those repositories sometimes mention using them for [mass account creation](https://github.com/lxlpp123/douyin-1), [automated likes](https://github.com/shenydowa/douyin-sign), and [crawling user information](https://github.com/shenydowa/douyin-sign). These capabilities may be further utilized to enable ad fraud, game the feed content curation algorithm, or worse, enable information operations (such as disinformation campaigns). In contrast with Douyin, we found fewer third-party X-Gorgon and ttEncrypt implementations specifically targeting TikTok. This difference could mean fewer bots are created to operate on TikTok. However, given the similarities between TikTok and Douyin’s APIs, bots created for Douyin may only require small modifications to work on TikTok.

Douyin’s Dynamic Code Loading
-----------------------------

### Douyin’s Plugin Framework

#### Initial Discovery

When testing Douyin on Feb 21, 2020, we noticed something unusual in its startup traffic (starting the app for the first time after installation, without any user interaction performed): there was a HTTP GET request towards https://lf6-ttcdn-tos.pstatp.com/obj/appeye/plugin/1128/caf79e812e33df0475b0e7ea9b282e10 `, which returned a APK file.

This behavior indicates that the app downloads code at startup, without any user interaction. From this observation, we suspect Douyin has the ability to load and run code downloaded via the Internet. Program features like this are referred to as _dynamic code loading_. Usually, when users launch a program, the operating system is in charge of loading the program’s code from its install location into the system memory. The program code would not change unless a newer version is installed. However, if a program has dynamic code loading capability, it can bypass this process.

There are a few common concerns regarding dynamic code loading:

1.  If the code downloading and loading mechanism contains any weakness, an attacker might be able to exploit this process to run malicious code on the device. Back in 2016, Citizen Lab researchers found such an instance in another Chinese app: [UC Browser for Windows and Android](https://citizenlab.ca/2016/08/a-tough-nut-to-crack-look-privacy-and-security-issues-with-uc-browser/).
2.  Even if the code downloading and loading mechanism contains no weakness, the dynamic code loading feature still allows the application to load code without notifying the user, bypassing users’ consent to decide what program could run on their device. For example, the developer may push out an unwanted update, and the users do not have a choice to keep using the old version.

To verify the downloaded APK is actually loaded and run, we searched for source code that performs loading and running APKs. We found the code package `com.bytedance.frameworks.plugin`, which includes a function `com.bytedance.frameworks.plugin.f.d.a`. The function uses a “stopwatch” utility function `com.bytedance.frameworks.plugin.p211h.C3465m.mo18300b()` to record the execution time of each block of code, and gives a name to each stopwatch. For convenience, here we use the stopwatch names to describe the actions performed in each block whenever possible (text in italics are stopwatch names):

1.  _checkSignature_
    1.  This code block uses Android system library’s [PackageManager.getPackageArchiveInfo()](https://developer.android.com/reference/android/content/pm/PackageManager#getPackageArchiveInfo(java.lang.String,%20int)) and getPackageInfo() to obtain the “plugin” APK’s cryptographic signature, and compare it with the signature of the current package (Douyin).
2.  _checkPermissions_
    1.  In this code block, Android’s system library function [PackageManager.getPackageArchiveInfo()](https://developer.android.com/reference/android/content/pm/PackageManager#getPackageArchiveInfo(java.lang.String,%20int)) and PackageManager.getPackageInfo() are called. The former call retrieves information about the downloaded APK file, the latter call retrieves information about the currently running application package. The code then further checks if all requested permissions in the downloaded APK are present in the current package. If not, an error with the message “安装包权限校验失败” (permission check failed for install package) will be raised.
3.  _copyApk_
    1.  Copy the downloaded APK files from their temporary location under /storage/emulated/0/Android/data/com.ss.android.ugc.aweme/files/.download/ into the app-specific storage under /data/data/com.ss.android.ugc.aweme/files/plugins/
4.  _copySo_
    1.  “So” likely refers to “Shared Object”, which is the kind of native library module file that can be loaded in an app.
    2.  Extracts the Shared Object files from the downloaded plugin APK into the main app-specific storage.
5.  Preparing and running dex2oat system command
    1.  [dex2oat](https://source.android.google.cn/devices/tech/dalvik/configure#how_art_works) is a system command that compiles code from APK files into runnable format.
6.  Instantiates com.bytedance.frameworks.plugin.core.g
    1.  com.bytedance.frameworks.plugin.core.g is a subclass of [dalvik.system.DexClassLoader](https://developer.android.com/reference/dalvik/system/DexClassLoader), which “loads classes from .jar and .apk files containing a classes.dex entry,” according to the documentation.
7.  _cleanPluginApk_
    1.  Delete the downloaded APK files from their temporary location.

[![](https://citizenlab.ca/wp-content/uploads/2021/03/img26.png)](https://citizenlab.ca/wp-content/uploads/2021/03/img26.png)

Figure 26: A portion of com.bytedance.frameworks.plugin.f.d.a source code.

In order to check if the com.bytedance.frameworks.plugin.f.d.a function was ever actually invoked, we used Frida to hook it. Whenever the function is invoked, the Frida hook will print the calling parameters on our console. We recorded the following calls:

*   `Apk{packageName=com.ss.android.ugc.aweme.videocache_v2, versionCode=1030, apkPath=/storage/emulated/0/Android/data/com.ss.android.ugc.aweme/files/.download/com.ss.android.ugc.aweme.videocache_v2.jar, installPriority=1}`
*   `Apk{packageName=com.ss.android.ugc.aweme.miniapp, versionCode=96004, apkPath=/storage/emulated/0/Android/data/com.ss.android.ugc.aweme/files/.download/com.ss.android.ugc.aweme.miniapp.jar, installPriority=1}`
*   `Apk{packageName=com.bytedance.aweme.f.two, versionCode=96005, apkPath=/storage/emulated/0/Android/data/com.ss.android.ugc.aweme/files/.download/com.bytedance.aweme.f.two.jar, installPriority=3}`

We pulled the files specified in “apkPath” from the device and matched them against the network requests we intercepted to find their source of download:

<table headertable="true"><thead><tr><th>apkPath</th><th>Original download URL</th><th>Checksum</th></tr></thead><tbody><tr><td><code>/storage/emulated/0/Android/data/com.ss.android.ugc.aweme/files/.download/com.ss.android.ugc.aweme.videocache_v2.jar</code></td><td>https://lf6-ttcdn-tos.pstatp.com/obj/appeye/plugin/1128/e85ba44093f3cf73073f62242859ce6a</td><td>e85ba44093f3cf73073f62242859ce6a</td></tr><tr><td><code>/storage/emulated/0/Android/data/com.ss.android.ugc.aweme/files/.download/com.ss.android.ugc.aweme.miniapp.jar</code></td><td>https://lf6-ttcdn-tos.pstatp.com/obj/appeye/plugin/1128/caf79e812e33df0475b0e7ea9b282e10</td><td>caf79e812e33df0475b0e7ea9b282e10</td></tr><tr><td><code>/storage/emulated/0/Android/data/com.ss.android.ugc.aweme/files/.download/com.bytedance.aweme.f.two.jar</code></td><td>https://lf6-ttcdn-tos.pstatp.com/obj/appeye/plugin/1128/71e7d3d08cd29572f74ffbe453fd8999</td><td>71e7d3d08cd29572f74ffbe453fd8999</td></tr></tbody></table>

Table 17: Path and download URL of Douyin’s plugin files.

We also found that, if the network is disconnected when we launch the app, the function com.bytedance.frameworks.plugin.f.d.a was never called.

We decompiled com.ss.android.ugc.aweme.miniapp.jar, one of the downloaded APK files. It contains a component called “Miniapp.” Ostensibly, it relates to [Bytedance’s “Micro App” functionality](https://microapp.bytedance.com/), which allows third-party developers to build “mini applications” using Javascript, HTML, and CSS. “Micro apps” can run on different Bytedance apps like Jinri Toutiao (今日头条) and Douyin.

[![](https://citizenlab.ca/wp-content/uploads/2021/03/img27.png)](https://citizenlab.ca/wp-content/uploads/2021/03/img27.png)

Figure 27: AndroidManifest.xml included in com.ss.android.ugc.aweme.miniapp.jar, which showed that it includes components such as “MiniAppListH5Activity.”

#### Invoking Dynamically Loaded Functions

So far we have shown Douyin downloads code and loads them through the system class loader, but were the loaded functions ever actually invoked? To find out, we extended our Frida script to hook functions declared in the dynamically loaded modules (plugins). We chose to [hook](https://github.com/citizenlab/tiktok-report-data/blob/main/frida-hook-miniapp.js) com.ss.android.ugc.aweme.miniapp.MiniAppService.inst() function from the plugin package com.ss.android.ugc.aweme.miniapp.jar. The result showed that com.ss.android.ugc.aweme.miniapp.MiniAppService.inst() was called, which confirmed that Douyin indeed run code downloaded via the internet.

### Douyin’s Unused Hot Patching Capability

Douyin contains [Meituan Robust](https://github.com/Meituan-Dianping/Robust), an open source library from Chinese company [Meituan](https://about.meituan.com/en). According to [Meituan](https://github.com/Meituan-Dianping/Robust), “Robust is an Android HotFix solution with high compatibility and high stability. Robust can fix bugs immediately without publishing apk.” Using the Robust library, apps can modify function implementations at runtime, for example allowing an updated version of the function to replace the existing copy without going through the system package installation process. In the rest of the section we discuss how Douyin integrated Robust.

In nearly every function within Douyin’s source code, we observed the following similar code block, invoking PatchProxy.isSupport() and PatchProxy.accessDispatch():

[![](https://citizenlab.ca/wp-content/uploads/2021/03/img28.png)](https://citizenlab.ca/wp-content/uploads/2021/03/img28.png)

Figure 28: Douyin’s hot patching hook functions.

isSupport() is defined within the package com.meituan.robust:

[![](https://citizenlab.ca/wp-content/uploads/2021/03/img29.png)](https://citizenlab.ca/wp-content/uploads/2021/03/img29.png)

Figure 29: isSupport() is defined within the package com.meituan.robust.

We studied Meituan Robust’s source code and [third-party write-ups](https://juejin.im/entry/6844903681276510222). Here we summarize how Meituan Robust works on a high level:

1.  At build time, functions are injected with isSupport() conditional check.
2.  At runtime, Meituan Robust maintains a list of “patched functions.” By calling com.meituan.robust.PatchExecutor.applyPatchList(), developers can specify the functions that should be replaced (patched) and what functions to replace them.
3.  Whenever isSupport() is called, it checks the list of “patched functions” to see whether the calling function has a replacement.
4.  If isSupport() determines the calling function needs to be replaced, it will return a value resulting in the calling function to call PatchProxy.accessDispatch(), which in turn calls the replacement function.

In order to see whether Meituan Robust was actually utilized, we tried hooking different functions used in various patching steps:

*   patch()
*   accessDispatch()

Then, we operated the app as such:

1.  Clear app data.
2.  Launch app.
3.  Accept Usage Policy prompt, deny phone calling permission and location permission.
4.  Wait for the update prompt.
5.  Click “Upgrade Immediately” (立即升级).
6.  Observe in our network traffic capture program, once the download completes, wait for at least 2 minutes.
7.  Terminate the app.

In our result after multiple tests, our Frida hooks were never triggered. This result suggests even though the Meituan Robust library is included in Douyin, it is not being used in practice. We are uncertain about why Meituan Robust was included but not used, or under what condition it would be used.

### Background App Updates

In our later rounds of testing the application (on August 27, 2020), we found that after launching the application,[9](#fn9) a message was prompted, saying “click download to update in the background” (点击下载，即可在后台更新):

[![](https://citizenlab.ca/wp-content/uploads/2021/03/img30.png)](https://citizenlab.ca/wp-content/uploads/2021/03/img30.png)

Figure 30: Douyin prompted to upgrade in the background.

After pressing “Upgrade Immediately” (立即升级), the prompt was closed and the user could continue using the app. No further message about upgrading was prompted.

In the background, we observed request to download an APK file:

[![](https://citizenlab.ca/wp-content/uploads/2021/03/img31.png)](https://citizenlab.ca/wp-content/uploads/2021/03/img31.png)

Figure 31: Douyin downloads an APK file upon clicking “Upgrade Immediately.”

We were not able to determine the update mechanism, as our attempt to trace the app update execution process all reached dead ends, potentially due to obfuscation. However, given 1) Douyin’s aforementioned “plugin framework” did not process this update APK and 2) our experiment with Meituan Robust suggested that it was not used, we can say that Douyin’s app updates rely on a mechanism independent of Meituan Robust and the plugin framework.

### Implications of Dynamic Code Loading

Dynamic code loading in Android is technically achievable, but a violation of Play Store policy. As laid out in [Google Play’s Developer Program Policy](https://web.archive.org/web/20200728154947/https://support.google.com/googleplay/android-developer/answer/9914283?hl=en):

An app distributed via Google Play may not modify, replace, or update itself using any method other than Google Play’s update mechanism. Likewise, an app may not download executable code (e.g. dex, JAR, .so files) from a source other than Google Play. This restriction does not apply to code that runs in a virtual machine and has limited access to Android APIs (such as JavaScript in a webview or browser).

This violation is probably one major reason why Douyin is not available in the Google Play Store.

Judging from [the abundance of hot patching support libraries](https://www.jianshu.com/p/6ae1e09ebbf5) developed by different Chinese companies, we think there are many more Chinese applications that include dynamic code loading capability. According to [some](http://zeatual.com/2018/08/02/plugin-framework-preface/) [Chinese](https://dim.red/2018/03/31/plugin_framework_%20exploration/) articles, hot patching application code is popular in China because:

1.  [Chinese market prefers monolithic apps](http://dangrover.com/blog/2014/12/01/chinese-mobile-app-ui-trends.html) (one app providing a whole suite of functionalities). To keep the app within a reasonable size so that it does not affect performance is difficult. Using dynamically loaded components is one way to minimize main app size.
2.  To work around issues caused by Chinese app stores. In China, the app stores market is fragmented, with each store having their own policies. The publishing approval duration also differs across different app stores.[10](#fn10) Having hot patching capability built into the app means that developers can directly push updates without going through app stores. It also does not require users to download APK files and install them manually.

Plugin Framework, Meituan Robust and App Updates Are Not in TikTok
------------------------------------------------------------------

The main program component of Douyin’s Plugin Framework, com.bytedance.frameworks.plugin, does not exist in Musically.

The TikTok source code can be divided into two parts: 1) ByteDance produced code and 2) third party code (library) that the app utilizes. In the first part, we did not find any occurrence of getSystemClassLoader() , which is a necessary function call in loading custom code. In the second part of the source code, getSystemClassLoader() does exist, but we deemed It to be unrelated to dynamic code loading. Note that TikTok may hide the call with code obfuscation.

The Meituan Robust framework does not exist in TikTok. We confirmed this by searching some of its code fragments across the entire Musically codebase. The isSupport() hooks used in Douyin functions were also never seen in Musically’s source code.

We also have not captured code-downloading traffic in Musically and Trill that are similar to the ones we found in Douyin.

Therefore, we think it is very unlikely that TikTok has the same dynamic code loading capability as Douyin.

We focused on two areas of censorship: post censorship and search censorship for TikTok and Douyin.

The evidence we found is inconclusive in showing if TikTok takes down posts (post censorship) unfavorable to the Chinese government. We found that TikTok contains a code module to handle a special class of server-returned search responses. If the server decides to restrict a search result, it will return a response with zero result item, and indicate the reason for restriction in a special “search_nil_info” field. The client code module contains four predefined reasons for restriction: “hate speech,” “limit,” “sensitive,” and “suicide prevention.” The server also sometimes returned reasons that were not predefined; however, we were unable to make sense of their meanings.

Douyin contains the same aforementioned module. However, when compared to TikTok, Douyin’s server was more restrictive. For many search keywords we tested, the Douyin server returned restricted results while TikTok returned normal results.

We performed searches on TikTok using a list of 5420 keyword combinations that have been previously found censored on WeChat and found that TikTok did not label any of the search results as “hate speech” or “sensitive.” From the same list, 392 of them were tested as search terms in Douyin. Out of those tested, 158 of them returned “restricted results.” Not all 5420 keyword combinations were tested because of Douyin’s API rate limiting.

Post censorship and search censorship are not the only forms of censorship that can exist in TikTok. One important form which we did not study is censorship in TikTok’s chat feature. And there are still more other forms of censorship, for example “shadowbanning,” which silently excludes censored posts from being seen by other users.

In the subsections below, we document our methodology and results for testing post censorship and search censorship.

Testing Post Censorship in TikTok
---------------------------------

Previously, it was reported that TikTok censored [politically sensitive](https://www.theguardian.com/technology/2019/sep/25/revealed-how-tiktok-censors-videos-that-do-not-please-beijing) and [LGBT-related](https://www.theguardian.com/technology/2019/sep/26/tiktoks-local-moderation-guidelines-ban-pro-lgbt-content) videos. A [report by Netzpolitik](https://netzpolitik.org/2019/cheerfulness-and-censorship/) showed that there were also different degrees of censorship, ranging from deleting the content entirely to reduced visibility. Information from these previous reports come from sources inside the company; however, little technical research has been conducted to detect censorship by interacting with the service directly. Therefore, we designed the following testing methodology to close this gap.

### Methodology

We focused on detecting deletion of user posts, since it is much more difficult to test for reduced visibility. We designed a two-phase “collect then check” process. In the first “collect” phase, we implemented a web crawler “Collector” to archive posts under a [set of hashtags](https://github.com/citizenlab/tiktok-report-data/tree/main/post_censorship_testing_hashtags) chosen by us. To crawl posts under a certain hashtag, the crawler simply visits [https://www.tiktok.com/tag/](https://tiktok.com/tag/)<HASHTAG> and extracts the post links and contents. The archived posts were recorded in our database, along with their timestamp and the status “up”.

In the second phase, we implemented “Checker,” which is also a web crawler, to check if previously archived posts are still available on the website. Checker will visit every collected post from the database and record status of posts at the time of visit. Status may be one of the following:

*   unavailable: the post page shows “Video currently unavailable” message
*   tag changed: the post text content no longer contains the keyword (tag) that was initially collected
*   program error: error during execution of the checker program.
*   private: the post was made private, and the page shows “This video is private”
*   up: up and all other conditions

Finally, we can identify posts that were taken down by tracking the status changes of each individual post. To rule out program errors and potential server errors in our detection, we only count a “change” if the new status is repeatedly recorded twice. For instance:

*   A post has 3 consecutive records of: up -> unavailable -> up : this is not counted as a change
*   A post has 3 consecutive records of: up -> unavailable -> unavailable : this is counted as a change from “up” to “unavailable”

Both crawlers ran every 24 hours between March 2 and April 20, 2020, excluding occasional down times of our system. In early June 2020, TikTok modified their website, making it much more difficult and time-consuming for our crawler to collect posts. After attempts to modify the crawler failed, we decided to stop the crawling and focus on analyzing the data that we collected.

Douyin has no web interface, so we could not use the same method as TikTok to test for post censorship. We presumed Douyin already censors user posts because [Douyin’s Terms of Service](https://archive.is/CqAkS) listed seven principles that all uploaded content should adhere to: laws, socialist system, national interest, legal rights of citizens, public order, morality, and truthfulness (法律法规、社会主义制度、国家利益、公民合法权益、社会公共秩序、道德风尚和信息真实性). If violated, the platform can delete the content or suspend the account. [TikTok’s Community Guidelines](https://web.archive.org/web/20201118060403/tiktok.com/community-guidelines) do not include similar requirements.

### Results

Across the entire testing period, we collected 28051 posts.[11](#fn11) Among those posts:

*   Group A: 722 of them were unavailable at some point during our testing. (They might come back up.)
*   Group B: 279 of them became unavailable and were still unavailable for the last three times we tested them. (Group B is a subset of Group A.)

The fact that some posts became unavailable is not definitive evidence that TikTok employs censorship, since we cannot rule out the possibility that the posts were taken down by the users themselves.

We analyzed the contents of Group B posts, specifically looking for:

*   Criteria 1: Contents that clearly violated TikTok’s Terms of Service, which might explain why they were taken down.
*   Criteria 2: Contents that are critical of the Chinese government, or show a negative image of China.

We did not find any post fitting Criteria 1. We found at least 16 posts fitting Criteria 2; however, for most of these posts, we could find similar content that was still available on the platform. One example of such is shown below:

[![](https://citizenlab.ca/wp-content/uploads/2021/03/img32.png)](https://citizenlab.ca/wp-content/uploads/2021/03/img32.png)

Figure 32: A post that became unavailable on March 5, 2020.

[![](https://citizenlab.ca/wp-content/uploads/2021/03/img33.png)](https://citizenlab.ca/wp-content/uploads/2021/03/img33.png)

Figure 33: A post in similar content that was still available as of December 22, 2020.

In conclusion, we think this evidence is insufficient to prove or disprove that TikTok employs political censorship. However, evidence does at least show that TikTok does not employ overt political censorship. In other words, TikTok does not censor solely based on the hashtags of a post. Political censorship, if it exists at all on TikTok, would be more nuanced. Finally, we lack better methodologies in detecting nuanced platform political censorship.

### Limitation

To crawl posts under a certain hashtag, the crawler visits https://www.tiktok.com/tag/<HASHTAG> . This page displays more popular posts first, and the user can scroll down to see more posts. Our crawler works like a robot-operated normal web browser. It was designed to visit the page, scroll to the page bottom 10 times, then start collecting individual posts. This would yield around 150 posts (per hashtag per crawl). The number of posts collected would fluctuate for two reasons:

1.  If the network condition was bad, after the crawler scrolls down, the page might not have sufficient time to load each individual post before the crawler collects it.
2.  If the post author’s user handle was not in alphanumeric characters, the link to that individual post would not be immediately available. In this situation, the crawler was designed to emulate a click on the post, this would load and reveal the link to that individual post. However, the loading time will also fluctuate depending on the network condition, and the crawler only waits for a fixed amount of time.

Usually, the total number of posts tagged with a hashtag would be far greater than 150, the number of posts we can collect on each run. And we could only collect the 150 most popular posts.

Our content analysis was restricted to the post text and video thumbnails. Our crawler did not archive full videos because of storage space restrictions.

In summary, the limitation to our post censorship testing methodology is that our sample only contains the most popular posts. This may affect our conclusion on whether post censorship was being enforced by the platform, because the platform may subtly only censor unpopular posts and leave out popular ones in order to avoid attention. However, given this limitation, at least we can say the platform does not enforce obvious post censorship, and if post censorship was enforced at all, it would subtly only apply to unpopular posts.

Search Censorship
-----------------

### Initial Discovery

We explored the source code intuitively by looking at Java class and package names,[12](#fn12) which led us to find the following symbols related to content restrictions.

[![](https://citizenlab.ca/wp-content/uploads/2021/03/img34.png)](https://citizenlab.ca/wp-content/uploads/2021/03/img34.png)

Figure 34: Symbols related to content restrictions in TikTok source code.

This class was used in the class “SearchApiResult”:

[![](https://citizenlab.ca/wp-content/uploads/2021/03/img35.png)](https://citizenlab.ca/wp-content/uploads/2021/03/img35.png)

Figure 35: Symbols related to content restrictions in TikTok source code.

SearchApiResult objects are returned by a HTTP API endpoint at https://api16-normal-c-alisg.tiktokv.com/aweme/v1/general/search/single/. This API endpoint is used whenever we perform searches in the “Discover” tab.

This finding shows that TikTok contains client side code to handle (parse) server returned “search result” responses. And that the response may contain labels that represent different kinds of empty search results. From the label names, it appears that reasons for empty search results may be:

*   HIT_HEAT_SPEECH: We think this is likely a misspell of “hate speech”
*   HIT_LIMIT
*   HIT_TYPE_SENSITIVE: We suspect this label means “politically sensitive,” but could not confirm.
*   suicide_prevent

We also found the same two classes SearchApiResult and SearchNilInfo in Douyin’s source code, under the same class path and with the same set of class property fields.

### Proof of Concept

We searched for “suicide” manually in Musically and it showed zero results on the interface.

[![](https://citizenlab.ca/wp-content/uploads/2021/03/img36.png)](https://citizenlab.ca/wp-content/uploads/2021/03/img36.png)

Figure 36: Searching for “suicide” in TikTok returned empty result.

In background, the search request is sent to https://api16-normal-c-useast1a.tiktokv.com/aweme/v1/general/search/single/?_rticket=1595281630350&op_region=CA&ts=1595281631& , using HTTP POST.

This was the response body:

```
{
   "status_code": 0,
   "data": [],
   "qc": "",
   "cursor": 0,
   "has_more": 0,
   "suicide_prevent": {
   "phone": "",
   "url": "https://www.tiktok.com/aweme/in_app/suicide/help/",
   "agent": ""
   },
   "tab_hide": 1,
   "ad_info": {},
   "extra": {
   "now": 1595281631000,
   "logid": "20200720214711010190185159455A1533",
   "fatal_item_ids": [],
   "search_request_id": ""
   },
   "log_pb": {
   "impr_id": "20200720214711010190185159455A1533"
   },
   "search_nil_info": 
   "search_nil_type": "",
   "is_load_more": "first_flush",
   "search_nil_item": "hit_self_harm",
   "text_type": 11
   },
   "guide_search_words": null,
   "backtrace": ""
} 


```

We also tested a few ethnic slurs using [the list from Wikipedia](https://en.wikipedia.org/wiki/List_of_ethnic_slurs) as search terms. The search returned empty results on the interface. And the server returned similar HTTP responses, with their “search_nil_item” specifying “hate_speech.”

From this, we have verified that SearchNilInfo does indeed result in an empty result page.

### Bulk Testing in TikTok

To test more search terms, we developed a program to programmatically perform searches using a list of keywords. The list we used was all censored [keyword combinations](https://github.com/citizenlab/tiktok-report-data/blob/main/wechat%20keywords-04-21-2020.csv) on Wechat found by our colleagues’ research from 2019/10/10 to 2020/04/21, containing 5420 keyword combinations. The keywords are mostly related to China’s politics and COVID-19. The test was started on April 22, 2020, and it took about 1.5 days to go through all keyword combinations. In the rest of this section, we document our process of developing and running this program.

Initially, we attempted to develop such a batch testing program by imitating the request sent by the app, but we soon found out that most API requests are protected by TikTok’s custom X-Gorgon HTTP header. Details on the X-Gorgon header are documented above in the “TikTok API Security” section. If we exclude the header in our request, the server will not return any search result. However, it was also difficult for us to reverse engineer the X-Gorgon generator module and re-implement the algorithm on our own.

Our second approach was to invoke the app’s internal function that performs searches, bypassing the graphical interface entirely. This technique is possible thanks to the Frida code instrumentation framework. With Frida, we can invoke arbitrary functions within the app and supply our own inputs to the function from the controlling computer. This way we do not need to generate the request (including the X-Gorgon header) from our program, since the invoked function in the app will call appropriate functions (including the X-Gorgon generator module) itself, just like a user performing normal search from the graphical interface. After invoking the search function, we use Frida’s Java.choose() function to search the app’s Java runtime heap (memory space) for the result object, then extract information from it. However, this approach has a few downsides, compared to the initial approach:

*   A rooted phone with Frida installed is required and the phone must be connected to the computer for Frida to control it. This requirement poses a limitation to parallelizing the search tests.
*   The app and phone runs with decreased stability. Both the app and the phone OS crash more often. This is expected because the way Frida works is by altering low level system program procedures, which naturally introduces instability. This is also why many of our searches failed, as described below.

We had success with the second approach. Out of all 5420 performed searches, the program captured 4685 responses. Those without a response were mostly due to program error (instability introduced by Frida) and network unavailability. Among all 4685 captured responses, there was only one combination that triggered SearchNilInfo response, and was not due to suicide prevention: “脑残 + 红十” (“retarded + red cross”).

To confirm, we searched the combination manually again, and the interface showed “no results.” We also recorded the HTTP response body:

```
{
   "status_code": 0,
   "data": [],
   "qc": "",
   "cursor": 0,
   "has_more": 0,
   "ad_info": {},
   "extra": 
   "now": 1587660663000
   "logid": "202004231651030101890730193E6D189F",
   "fatal_item_ids": [], 
   "search_request_id": ""
   },
   "log_pb": {
   "impr_id": "202004231651030101890730193E6D189F"
   },
   "search_nil_info": 
   "search_nil_type": "",
   "is_load_more": "first_flush"
   "search_nil_item": "pack_nil"
   "text_type": 10
   },
   "guide_search_words": null
}


```

The response body shows that the keyword combination did indeed trigger SearchNilInfo response. But interestingly, the “reason” indicated by the “search_nil_item” field is novel and never seen in client source code: “pack_nil.” We could not find any occurrences of the string in the source code, therefore we are uncertain of its meaning.

In our retest on 2020 September 25, “脑残 + 红十” returned search results normally.

In summary, other than “脑残 + 红十” (“retarded + red cross”) and 3 self-harm keyword combinations:

*   自杀未遂 + 杰瑞 + 性侵 + 警方 (attempted suicide + Jerry + sexual assault + police)
*   股民 + 跳樓 + 大大 (stockholders + jumped + big)
*   林子健 + 被擄走 + 自殘 (Lin Zijian + abducted + self-harm)

All other keywords returned non-empty search results. Because none of the politically sensitive search terms returned empty search results, we conclude that it is unlikely TikTok employs political censorship in the search feature.

### Bulk Testing in Douyin

We then shifted our focus to test for search censorship on Douyin. Thanks to their code similarity, we only needed to slightly modify our automated search program for TikTok to make it work for Douyin.

We supplied the modified search program with the same list we used to test TikTok. We executed this program between May 1 and May 5, 2020. We soon found out that Douyin appeared to be stricter than TikTok on blocking automated requests. After around 400 searches,[13](#fn13) Douyin’s API server started returning a Chinese error message, saying “please retry later.”

Out of all search attempts, our program captured 392 responses. Among these captured responses, 158 of them included the “search_nil_item” field, indicating that the results were restricted.

The full list of restricted keyword combinations we found is documented on [GitHub](https://github.com/citizenlab/tiktok-report-data/blob/main/douyin_search_restricted.csv). Following are a few examples of restricted keyword combinations, with their “search_nil_item” field:

<table headertable="true"><thead><tr><th><strong>Original search term</strong></th><th><strong>English Translation</strong></th><th><strong>“search_nil_item” field value</strong></th></tr></thead><tbody><tr><td>反修例示威 + 一國兩制 + 自毀長城 + 時事評論員</td><td>Anti-revision demonstration + one country, two systems + self-destructing the Great Wall + current affairs commentator</td><td>federation_empty</td></tr><tr><td>法律 + 立法會 + 中央 + 駱惠寧</td><td>Law + Legislative Council + Central Government + Luo Huining</td><td>federation_empty</td></tr><tr><td>肺炎 + 中央 + 防疫 + 病毒 + 台灣</td><td>Pneumonia + Central Government + Epidemic Prevention + Virus + Taiwan</td><td>hit_dirt_words</td></tr><tr><td>无症状 + 确诊病例 + 中共 + 综合</td><td>Asymptomatic + confirmed case + CCP + comprehensive</td><td>hit_dirt_words</td></tr><tr><td>习近平 + 新高地</td><td>Xi Jinping + New Heights</td><td>federation_empty</td></tr></tbody></table>

Table 18: Examples of restricted search keyword combinations in Douyin.

The values “federation_empty” and “hit_dirt_words” were not present in Douyin’s source code, so we are uncertain of their meanings.

Following are a few examples of unrestricted keyword combinations:

<table headertable="true"><thead><tr><th><strong>Original search term</strong></th><th><strong>English Translation</strong></th></tr></thead><tbody><tr><td>撑不住 + 主席 + 香港</td><td>Unable to hold + Chairman + Hong Kong</td></tr><tr><td>干预香港 + 基本法 + 主席 + 立法会</td><td>Intervention in Hong Kong + Basic Law + Chairman + Legislative Council</td></tr><tr><td>台灣 + 武漢 + 肺炎 + 中央</td><td>Taiwan + Wuhan + Pneumonia + Central</td></tr><tr><td>无症状 + 确诊病例 + 公交车 + 病毒</td><td>Asymptomatic + confirmed case + bus + virus</td></tr><tr><td>质疑 + 湖北人 + 大帝</td><td>Questioning + Hubei people + Great Emperor</td></tr></tbody></table>

Table 19: Examples of unrestricted search keyword combinations in Douyin.

In summary, we have shown that Douyin does restrict certain politically sensitive terms in their search results. However, our current methodology only took what is known to be censored on WeChat and tested it in Douyin, we did not attempt to discover new censored keywords. We are also uncertain of the exact text-matching conditions which triggers an empty search result, for example, does the sequence of words and characters matter? does whitespaces matter? Further research is needed to answer these questions.

This project was motivated by the following questions: Did ByteDance adapt its social media platform to the international market in alignment with industry norms? Or did it implement or retain some features that are required in the Chinese market that may present undesirable security and privacy risks to non-mainland China users?

From the similarities and differences we have found among TikTok and Douyin, it appears that ByteDance’s strategy is to have a common code base and then to customize it to fit different market requirements. This strategy makes sense from an economic perspective. The distinction of TikTok’s two variants, one for East Asia and Southeast Asia, the other for the rest of the world, shows that ByteDance reuses this strategy to further divide the needs of different regions. This distinction of Trill, which targets East Asia and Southeast Asia, versus Musically, which targets everywhere else, also further shows ByteDance’s emphasis on East Asia and Southeast Asia.

Throughout our research we observe this strategy being put to use. These examples we found also showed the various levels of customization being applied to the common code base. Customization such as choosing different log servers is implemented with an internal configuration value in a common code base. Some customizations are dictated by server-returned values, such as the “share to Facebook Story” feature. Feature modules that may attract scrutiny are entirely removed, for example the dynamic code loading support modules are only present in Douyin, but not TikTok. Another class of customizations can be implemented entirely server-side, such as search censorship. Even though the search censorship modules in TikTok and Douyin predefined the same set of restrictions that can be applied, Douyin applied these restrictions differently from TikTok by returning different values from the server.

From the cases we have found, these customizations seem to be applied appropriately. In TikTok, the end result of customizing the common code base seems to create a product that largely follows international industry norms, as we have not found any undesirable features like the ones in Douyin, nor strong deviations of privacy, security and censorship practices when compared to TikTok’s competitors, like Facebook.

However, TikTok is also not completely free of privacy, security, and censorship concerns. . As we have shown, ByteDance relies on these different degrees of customizations of the common code base to tailor to different market needs. These customizations might be reverted with varying degrees of difficulty, whether intentionally or unintentionally. For example, it would be relatively difficult to enable dynamic code loading in TikTok, since it does not contain necessary code to aid the process. On the other hand, we have shown that TikTok does contain some dormant code originally written for Douyin, and TikTok’s server-returned configuration values do dictate some of its behaviors. We are concerned with the possibility where TikTok’s server-returned configuration values could enable those dormant code written for Douyin, which might lead to China-specific features being enabled.

Open Technology Fund funded this research through the [Information Controls Fellowship Program](https://www.opentech.fund/funds/icfp/). Graphic design by Mari Zhou. Special thanks to Jeffrey Knockel, Miles Kenyon, Masashi Nishihata, Lotus Ruan, and Adam Senft. Project supervised by Ron Deibert, Citizen Lab Director.

[https://github.com/citizenlab/tiktok-report-data](https://github.com/citizenlab/tiktok-report-data)