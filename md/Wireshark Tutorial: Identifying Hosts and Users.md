> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [unit42.paloaltonetworks.com](https://unit42.paloaltonetworks.com/using-wireshark-identifying-hosts-and-users/)

![](https://unit42.paloaltonetworks.com/wp-content/uploads/2019/03/tutorial-1-900x512.png)

This post is also available in: [日本語 (Japanese)](https://unit42.paloaltonetworks.jp/using-wireshark-identifying-hosts-and-users/)

When a host is infected or otherwise compromised, security professionals need to quickly review packet captures (pcaps) of suspicious network traffic to identify affected hosts and users.

This tutorial offers tips on how to gather that pcap data using Wireshark, the widely used network protocol analysis tool. It assumes you understand network traffic fundamentals and will use [these](https://www.malware-traffic-analysis.net/training/host-and-user-ID.html) pcaps of [IPv4](https://en.wikipedia.org/wiki/IPv4) traffic to cover retrieval of four types of data:

*   Host information from DHCP traffic
*   Host information from NetBIOS Name Service (NBNS) traffic
*   Device models and operating systems from HTTP traffic
*   Windows user account from Kerberos traffic

Host Information from DHCP Traffic
----------------------------------

Any host generating traffic within your network should have three identifiers: a [MAC address](https://en.wikipedia.org/wiki/MAC_address), an [IP address](https://en.wikipedia.org/wiki/IP_address), and a [hostname](https://en.wikipedia.org/wiki/Hostname).

In most cases, alerts for suspicious activity are based on IP addresses. If you have access to full packet capture of your network traffic, a pcap retrieved on an internal IP address should reveal an associated MAC address and hostname.

How do we find such host information using Wireshark? We filter on two types of activity: [DHCP](https://wiki.wireshark.org/DHCP) or [NBNS](https://wiki.wireshark.org/NetBIOS/NBNS). DHCP traffic can help identify hosts for almost any type of computer connected to your network. NBNS traffic is generated primarily by computers running Microsoft Windows or Apple hosts running MacOS.

The first pcap for this tutorial, **_host-and-user-ID-pcap-01.pcap_**, is available [here](https://www.malware-traffic-analysis.net/training/host-and-user-ID.html). This pcap is for an internal IP address at 172.16.1[.]207. Open the pcap in Wireshark and filter on **_bootp_** as shown in Figure 1. This filter should reveal the DHCP traffic.

**Note**: With Wireshark 3.0, you must use the search term **dhcp** instead of **bootp**.

[![](https://unit42.paloaltonetworks.com/wp-content/uploads/2020/03/Figure-1-Filtering-on-DHCP-traffic-in-Wireshark.png)](https://unit42.paloaltonetworks.com/wp-content/uploads/2020/03/Figure-1-Filtering-on-DHCP-traffic-in-Wireshark.png)

_Figure 1: Filtering on DHCP traffic in Wireshark_

Select one of the frames that shows **_DHCP Request_** in the info column. Go to the frame details section and expand the line for **_Bootstrap Protocol (Request)_** as shown in Figure 2. Expand the lines for **_Client Identifier_** and **_Host Name_** as indicated in Figure 3. Client Identifier details should reveal the MAC address assigned to 172.16.1[.]207, and Host Name details should reveal a hostname.

[![](https://unit42.paloaltonetworks.com/wp-content/uploads/2020/03/Figure-2-Expanding-Bootstrap-Protocol-line-from-a-DHCP-request..png)](https://unit42.paloaltonetworks.com/wp-content/uploads/2020/03/Figure-2-Expanding-Bootstrap-Protocol-line-from-a-DHCP-request..png)

_Figure 2: Expanding Bootstrap Protocol line from a DHCP request_

[![](https://unit42.paloaltonetworks.com/wp-content/uploads/2020/03/Figure-3-Finding-the-MAC-address-and-hostname-in-a-DHCP-request..png)](https://unit42.paloaltonetworks.com/wp-content/uploads/2020/03/Figure-3-Finding-the-MAC-address-and-hostname-in-a-DHCP-request..png)

_Figure 3: Finding the MAC address and hostname in a DHCP request_

In this case, the hostname for 172.16.1[.]207 is **_Rogers-iPad_** and the MAC address is **_7c:6d:62:d2:e3:4f_**. This MAC address is assigned to Apple. Based on the hostname, this device is likely an iPad, but we cannot confirm solely on the hostname.

We can easily correlate the MAC address and IP address for any frame with 172.16.1[.]207 as shown in Figure 4.

[![](https://unit42.paloaltonetworks.com/wp-content/uploads/2020/03/Figure-4-Correlating-the-MAC-address-with-the-IP-address-from-any-frame..png)](https://unit42.paloaltonetworks.com/wp-content/uploads/2020/03/Figure-4-Correlating-the-MAC-address-with-the-IP-address-from-any-frame..png)

_Figure 4: Correlating the MAC address with the IP address from any frame_

Host Information from NBNS Traffic
----------------------------------

Depending on how frequently a DHCP lease is renewed, you might not have DHCP traffic in your pcap. Fortunately, we can use NBNS traffic to identify hostnames for computers running Microsoft Windows or Apple hosts running MacOS.

The second pcap for this tutorial, **_host-and-user-ID-pcap-02.pcap_**, is available [here](https://www.malware-traffic-analysis.net/training/host-and-user-ID.html). This pcap is from a Windows host using an internal IP address at 10.2.4[.]101. Open the pcap in Wireshark and filter on **_nbns_**. This should reveal the NBNS traffic. Select the first frame, and you can quickly correlate the IP address with a MAC address and hostname as shown in Figure 5.

[![](https://unit42.paloaltonetworks.com/wp-content/uploads/2020/03/Figure-5-Correlating-hostname-with-IP-and-MAC-address-using-NBNS-traffic..png)](https://unit42.paloaltonetworks.com/wp-content/uploads/2020/03/Figure-5-Correlating-hostname-with-IP-and-MAC-address-using-NBNS-traffic..png)

_Figure 5: Correlating hostname with IP and MAC address using NBNS traffic_

The frame details section also shows the hostname assigned to an IP address as shown in Figure 6.

[![](https://unit42.paloaltonetworks.com/wp-content/uploads/2020/03/Figure-6-Frame-details-for-NBNS-traffic-showing-the-hostname-assigned-to-an-IP-address..png)](https://unit42.paloaltonetworks.com/wp-content/uploads/2020/03/Figure-6-Frame-details-for-NBNS-traffic-showing-the-hostname-assigned-to-an-IP-address..png)

_Figure 6: Frame details for NBNS traffic showing the hostname assigned to an IP address_

Device Models and Operating Systems from HTTP Traffic
-----------------------------------------------------

User-agent strings from headers in HTTP traffic can reveal the operating system. If the HTTP traffic is from an Android device, you might also determine the manufacturer and model of the device.

The third pcap for this tutorial, **_host-and-user-ID-pcap-03.pcap_**, is available [here](https://www.malware-traffic-analysis.net/training/host-and-user-ID.html). This pcap is from a Windows host using an internal IP address at 192.168.1[.]97. Open the pcap in Wireshark and filter on **_http.request and !(ssdp)_**. Select the second frame, which is the first HTTP request to **_www.ucla[.]edu_**, and follow the TCP stream as shown in Figure 7.

[![](https://unit42.paloaltonetworks.com/wp-content/uploads/2020/03/Figure-7-Following-the-TCP-stream-for-an-HTTP-request-in-the-third-pcap..png)](https://unit42.paloaltonetworks.com/wp-content/uploads/2020/03/Figure-7-Following-the-TCP-stream-for-an-HTTP-request-in-the-third-pcap..png)

_Figure 7: Following the TCP stream for an HTTP request in the third pcap_

This TCP stream has HTTP request headers as shown in Figure 8. The User-Agent line represents Google Chrome web browser version 72.0.3626[.]81 running on Microsoft's Windows 7 x64 operating system.

[![](https://unit42.paloaltonetworks.com/wp-content/uploads/2020/03/Figure-8-The-User-Agent-line-for-a-Windows-7-x64-host-using-Google-Chrome..png)](https://unit42.paloaltonetworks.com/wp-content/uploads/2020/03/Figure-8-The-User-Agent-line-for-a-Windows-7-x64-host-using-Google-Chrome..png)

_Figure 8: The User-Agent line for a Windows 7 x64 host using Google Chrome_

_Note the following string in the User-Agent line from Figure 8:_

**_(Windows NT 6.1; Win64; x64)_**

**_Windows NT 6.1_** represents Windows 7. For User-Agent lines, Windows NT strings represent the following versions of Microsoft Windows as shown below:

*   Windows NT 5.1: **_Windows XP_**
*   Windows NT 6.0: **_Windows Vista_**
*   Windows NT 6.1: **_Windows 7_**
*   Windows NT 6.2: **_Windows 8_**
*   Windows NT 6.3: **_Windows 8.1_**
*   Windows NT 10.0: **_Windows 10_**

With HTTP-based web browsing traffic from a Windows host, you can determine the operating system and browser. The same type of traffic from Android devices can reveal the brand name and model of the device.

The fourth pcap for this tutorial, **_host-and-user-ID-pcap-04.pcap_**, is available [here](https://www.malware-traffic-analysis.net/training/host-and-user-ID.html). This pcap is from an Android host using an internal IP address at 172.16.4.119. Open the pcap in Wireshark and filter on **_http.request_**. Select the second frame, which is the HTTP request to **_www.google[.]com_** for **_/blank.html_**. Follow the TCP stream as shown in Figure 9.

[![](https://unit42.paloaltonetworks.com/wp-content/uploads/2020/03/Figure-9-Following-the-TCP-stream-for-an-HTTP-request-in-the-fourth-pcap..png)](https://unit42.paloaltonetworks.com/wp-content/uploads/2020/03/Figure-9-Following-the-TCP-stream-for-an-HTTP-request-in-the-fourth-pcap..png)

_Figure 9: Following the TCP stream for an HTTP request in the fourth pcap_

[![](https://unit42.paloaltonetworks.com/wp-content/uploads/2020/03/Figure-10-The-User-Agent-line-for-an-Android-host-using-Google-Chrome..png)](https://unit42.paloaltonetworks.com/wp-content/uploads/2020/03/Figure-10-The-User-Agent-line-for-an-Android-host-using-Google-Chrome..png)

_Figure 10: The User-Agent line for an Android host using Google Chrome_

The User-Agent line in Figure 10 shows **_Android 7.1.2_** which is an older version of the Android operating system released in April 2017. **_LM-X210APM_** represents a model number for this Android device. A [quick Google search](https://www.google.com/search?q=LM-X210APM) reveals this model is an [LG Phoenix 4 Android smartphone](https://www.lg.com/us/support-mobile/lg-LMX210APM).

The User-Agent line for HTTP traffic from an iPhone or other Apple mobile device will give you the operating system, and it will give you the type of device. However, it will not give you a model. We can only determine if the Apple device is an iPhone, iPad, or iPod. We cannot determine the model.

The fifth pcap for this tutorial, **_host-and-user-ID-pcap-05.pcap_**, is available [here](https://www.malware-traffic-analysis.net/training/host-and-user-ID.html). This pcap is from an iPhone host using an internal IP address at 10.0.0[.]114. Open the pcap in Wireshark and filter on **_http.request_**. Select the frame for the first HTTP request to **_web.mta[.]info_** and follow the TCP stream as shown in Figure 11.

[![](https://unit42.paloaltonetworks.com/wp-content/uploads/2020/03/Figure-11-Following-the-TCP-stream-for-an-HTTP-request-in-the-fifth-pcap..png)](https://unit42.paloaltonetworks.com/wp-content/uploads/2020/03/Figure-11-Following-the-TCP-stream-for-an-HTTP-request-in-the-fifth-pcap..png)

_Figure 11: Following the TCP stream for an HTTP request in the fifth pcap_

In Figure 12, the User-Agent line shows **_(iPhone; CPU iPhone OS 12_1_3 like Mac OS X)_**. This indicates the Apple device is an iPhone, and it is running iOS 12.1.3.

[![](https://unit42.paloaltonetworks.com/wp-content/uploads/2020/03/Figure-12-The-User-Agent-line-for-an-iPhone-using-Safari..png)](https://unit42.paloaltonetworks.com/wp-content/uploads/2020/03/Figure-12-The-User-Agent-line-for-an-iPhone-using-Safari..png)

_Figure 12: The User-Agent line for an iPhone using Safari_

A final note about HTTP traffic and User-Agent strings: not all HTTP activity is web browsing traffic. Some HTTP requests will not reveal a browser or operating system. When you search through traffic to identify a host, you might have to try several different HTTP requests before finding web browser traffic.

Since more websites are using HTTPS, this method of host identification can be difficult. HTTP headers and content are not visible in HTTPS traffic. However, for those lucky enough to find HTTP web-browsing traffic during their investigation, this method can provide more information about a host.

Windows User Account from Kerberos Traffic
------------------------------------------

For Windows hosts in an Active Directory (AD) environment, we can find user account names in from Kerberos traffic.

The sixth pcap for this tutorial, **_host-and-user-ID-pcap-06.pcap_**, is available [here](https://www.malware-traffic-analysis.net/training/host-and-user-ID.html). This pcap is from a Windows host in the following AD environment:

*   Domain: **_happycraft[.]org_**
    *   Network segment: **_172.16.8.0/24_** (172.16.8[.]0 - 172.16.8[.]255)
*   Domain controller IP: **_172.16.8[.]8_**
*   Domain controller hostname: **_Happycraft-DC_**
*   Segment gateway: **_172.16.8[.]1_**
*   Broadcast address: **_172.16.8[.]255_**
*   Windows client: **_172.16.8[.]201_**

Open the pcap in Wireshark and filter on **_kerberos.CNameString_**. Select the first frame. Go to the frame details section and expand lines as shown in Figure 13. Select the line with **_CNameString: johnson-pc$_** and apply it as a column.

[![](https://unit42.paloaltonetworks.com/wp-content/uploads/2020/03/Figure-13-Finding-the-CNameString-value-and-applying-it-as-a-column..png)](https://unit42.paloaltonetworks.com/wp-content/uploads/2020/03/Figure-13-Finding-the-CNameString-value-and-applying-it-as-a-column..png)

_Figure 13: Finding the CNameString value and applying it as a column_

This should create a new column titled **_CNameString_**. Scroll down to the last frames in the column display. You should find a user account name for **_theresa.johnson_** in traffic between the domain controller at 172.16.8[.]8 and the Windows client at 172.16.8[.]201 as shown in Figure 14.

[![](https://unit42.paloaltonetworks.com/wp-content/uploads/2020/03/Figure-14-Finding-the-W.png)](https://unit42.paloaltonetworks.com/wp-content/uploads/2020/03/Figure-14-Finding-the-W.png)

_Figure 14: Finding the Windows user account name_

CNameString values for hostnames always end with a **_$_** (dollar sign), while user account names do not. To filter on user account names, use the following Wireshark expression to eliminate CNameString results with a dollar sign:

**_kerberos.CNameString and !(kerberos.CNameString contains $)_**

Summary
-------

Proper identification of hosts and users from network traffic is essential when reporting malicious activity in your network. Using the methods from this tutorial, we can better utilize Wireshark to help us identify affected hosts and users.

For more help using Wireshark, please see our previous tutorials:

*   [Customizing Wireshark – Changing Your Column Display](https://unit42.paloaltonetworks.com/unit42-customizing-wireshark-changing-column-display/)
*   [Using Wireshark: Display Filter Expressions](https://unit42.paloaltonetworks.com/using-wireshark-display-filter-expressions/)

#### Get updates from  
Palo Alto  
Networks!

Sign up to receive the latest news, cyber threat intelligence and research from us