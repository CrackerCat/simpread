> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [unit42.paloaltonetworks.com](https://unit42.paloaltonetworks.com/wireshark-tutorial-examining-trickbot-infections/)

![](https://unit42.paloaltonetworks.com/wp-content/uploads/2019/11/tutorial-900x512-900x512.png)

This post is also available in: [日本語 (Japanese)](https://unit42.paloaltonetworks.jp/wireshark-tutorial-examining-trickbot-infections/)

**Executive Summary**
---------------------

When a host is infected or otherwise compromised, security professionals with access to packet captures (pcaps) of the network traffic need to understand the activity and identify the type of infection.

This tutorial offers tips on how to identify Trickbot, an information stealer and banking malware that has been [infecting victims since 2016](https://www.forbes.com/sites/leemathews/2019/07/14/stealthy-trickbot-malware-has-compromised-250-million-email-accounts-and-is-still-going-strong/#6f92d6d94884). Trickbot is distributed through malicious spam (malspam), and it is also distributed by other malware such as [Emotet](https://www.malware-traffic-analysis.net/2019/10/02/index.html), [IcedID](https://www.malware-traffic-analysis.net/2019/10/31/index.html), or [Ursnif](https://www.malware-traffic-analysis.net/2019/10/09/index.html).

Trickbot has distinct traffic patterns. This tutorial reviews pcaps of Trickbot infections caused by two different methods: a Trickbot infection from malspam and Trickbot when it is distributed through other malware.

Note: Today’s tutorial requires Wireshark with a column display customized according to [this previous tutorial](https://unit42.paloaltonetworks.com/unit42-customizing-wireshark-changing-column-display/). You should already have implemented Wireshark display filters as described [here](https://unit42.paloaltonetworks.com/using-wireshark-display-filter-expressions/).

#### **Trickbot from malspam**

Trickbot is often distributed through malspam. Emails from these campaigns contain links to download malicious files disguised as invoices or documents. These files may be Windows executable files for Trickbot, or they may be some sort of downloader for the Trickbot executable. In some cases, links from these emails return a zip archive that contains a Trickbot executable or downloader.

Figure 1 shows an example from September 2019. In this example, the email contained a link that returned a zip archive. The zip archive contained a Windows shortcut file that downloaded a Trickbot executable. A pcap for the associated Trickbot infection is available [here](https://www.malware-traffic-analysis.net/2019/09/25/index.html).

![](https://unit42.paloaltonetworks.com/wp-content/uploads/2019/11/word-image-61-1024x554.png)  
_Figure 1: Flowchart from a Trickbot infection from malspam in September 2019._

Download the pcap [from this page](https://www.malware-traffic-analysis.net/2019/09/25/index.html). The pcap is contained in a password-protected zip archive named **_2019-09-25-Trickbot-gtag-ono19-infection-traffic.pcap.zip_**. Extract the pcap from the zip archive using the password **_infected_** and open it in Wireshark. Use your **_basic_** filter to review the web-based infection traffic as shown in Figure 2.

_![](https://unit42.paloaltonetworks.com/wp-content/uploads/2019/11/word-image-63-1024x570.png)Figure 2: Pcap of the Trickbot infection viewed in Wireshark._

Review the traffic, and you will find the following activity common in recent Trickbot infections:

*   An IP address check by the infected Windows host
*   HTTPS/SSL/TLS traffic over TCP ports 447 and 449
*   HTTP traffic over TCP port 8082
*   HTTP requests ending in **_.png_** that return Windows executable files

Unique to this Trickbot infection is an HTTP request to **_www.dchristjan[.]com_** that returned a zip archive and an HTTP request to **_144.91.69[.]195_** that returned a Windows executable file. Follow the HTTP stream for the request to **_www.dchristjan[.]com_** as shown in Figure 3 to review the traffic. In the HTTP stream, you will find indicators that a zip archive was returned as shown in Figure 4.

![](https://unit42.paloaltonetworks.com/wp-content/uploads/2019/11/word-image-65-1024x580.png)_Figure 3: Following the HTTP stream for the request to www.dchristjan[.]com._

_![](https://unit42.paloaltonetworks.com/wp-content/uploads/2019/11/word-image-67-1024x796.png)Figure 4: Indicators the HTTP request returned a zip archive._

In Figure 4, you can also see the name of the file contained in the zip archive, **_InvoiceAndStatement.lnk_**. You can export the zip archive from the traffic using Wireshark as shown in Figure 5 and Figure 6 using the following path:

 **_File → Export Objects → HTTP…_**

_![](https://unit42.paloaltonetworks.com/wp-content/uploads/2019/11/word-image-69-1024x573.png)Figure 5: Exporting HTTP objects from the pcap._

_![](https://unit42.paloaltonetworks.com/wp-content/uploads/2019/11/word-image-70-1024x477.png)  
Figure 6: Exporting the zip archive from the pcap._

In a BSD, Linux, or Mac environment, you can easily confirm the extracted file is a zip archive, get the SHA256 hash of the file, and extract the contents of the archive in a command line environment. In this case, the content is a Windows shortcut file, which you can also confirm and get the SHA256 hash as shown in Figure 7.

The command to identify the file type is **file** _[filename]_, while the command to find the SHA256 hash of the file is **shasum -a 256** _[filename]_.

_![](https://unit42.paloaltonetworks.com/wp-content/uploads/2019/11/word-image-72-1024x520.png)Figure 7: Checking the extracted zip archive and its contents._

An HTTP request to **_144.91.69[.]195_** returned a Windows executable file. This is the initial Windows executable for Trickbot. You can follow the HTTP stream for this HTTP request and find indicators this is an executable file as shown in Figure 8 and Figure 9. You can extract the executable file from the pcap as shown in Figure 10.

_![](https://unit42.paloaltonetworks.com/wp-content/uploads/2019/11/word-image-73-1024x706.png)Figure 8: Following the HTTP stream for the HTTP request to 144.91.69[.]195._

_![](https://unit42.paloaltonetworks.com/wp-content/uploads/2019/11/word-image-74-1024x755.png)Figure 9: Indicators the returned file is a Windows executable or DLL file._

_![](https://unit42.paloaltonetworks.com/wp-content/uploads/2019/11/word-image-75-1024x546.png)Figure 10: Exporting the Windows executable from the pcap._

Post infection traffic initially consists of HTTPS/SSL/TLS traffic over TCP port 443, 447, or 449 and an IP address check by the infected Windows host. In this infection, shortly after the HTTP request for the Trickbot executable, we can see several attempted TCP connections over port 443 to different IP addresses before the successful TCP connection to 187.58.56[.]26 over TCP port 449. If you use your **_basic+_** filter, you can see these attempted connections as shown in Figure 11 and Figure 12.

_![](https://unit42.paloaltonetworks.com/wp-content/uploads/2019/11/word-image-76-1024x704.png)Figure 11: Attempted TCP connections over port 443 by the infected Windows host._

_![](https://unit42.paloaltonetworks.com/wp-content/uploads/2019/11/word-image-77-1024x705.png)Figure 12: Scrolling down to see more TCP connections over port 443 before a successful connection to 187.58.56[.]26 over TCP port 449._

The HTTPS/SSL/TLS traffic to various IP addresses over TCP port 447 and TCP port 449 has unusual certificate data. We can review the certificate issuer by filtering on **_ssl.handshake.type == 11_** when using Wireshark 2.x or **_tls.handshake.type == 11_** when using Wireshark 3.x. Then go to the frame details section and expand the information, finding your way to the certificate issuer data as seen in Figure 13 and Figure 14.

![](https://unit42.paloaltonetworks.com/wp-content/uploads/2019/11/word-image-78-1024x706.png)_Figure 13: Filtering for the certificate data in the HTTPS/SSL/TLS traffic, then expanding lines the frame details for the first result under TCP port 449._

_![](https://unit42.paloaltonetworks.com/wp-content/uploads/2019/11/word-image-79-1024x704.png)Figure 14: Drilling down to the certificate issuer data on the first result over TCP port 449._

In Figure 14, we see the following certificate issuer data used in HTTPS/SSL/TLS traffic to 187.58.56[.]26 over TCP port 449:

*   id-at-countryName=**_AU_**
*   id-at-stateOrProvinceName=**_Some-State_**
*   id-at-organizationName=**_Internet Widgits Pty Ltd_**

The state or province name (Some-State) and the organization name (Internet Widgits Pty Ltd) are not used for legitimate HTTPS/SSL/TLS traffic. This is an indicator of malicious traffic, and this type of unusual certificate issuer data is not limited to Trickbot. What does a normal certificate issuer look like in legitimate HTTPS/SSL/TLS traffic? If we look at earlier traffic to Microsoft domains at 72.21.81.200 over TCP port 443, we find the following as seen in Figure 15.

*   id-at-countryName=**_US_**
*   id-at-stateOrProvinceName=**_Washington_**
*   id-at-localityName=**_Redmond_**
*   id-at-organizationName=**_Microsoft Corporation_**
*   id-at-organizationUnitName=**_Microsoft IT_**
*   id-at-commonName=**_Microsoft IT TLS CA 2_**

![](https://unit42.paloaltonetworks.com/wp-content/uploads/2019/11/word-image-80-1024x586.png)_Figure 15: Certificate data from legitimate HTTPS traffic to a Microsoft domain._

The Trickbot-infected Windows host will check its IP address using a number of different IP address checking sites. These sites are **_not_** malicious, and the traffic is not inherently malicious. However, this type of IP address check is common with Trickbot and other families of malware. Various legitimate IP address checking services used by Trickbot include:

*   **_api.ip[.]sb_**
*   **_checkip.amazonaws[.]com_**
*   **_icanhazip[.]com_**
*   **_ident[.]me_**
*   **_ip.anysrc[.]net_**
*   **_ipecho[.]net_**
*   **_ipinfo[.]io_**
*   **_myexternalip[.]com_**
*   **_wtfismyip[.]com_**

Again, an IP address check by itself is not malicious. However, this type of activity combined with other network traffic can provide indicators of an infection, like we see in this case.

![](https://unit42.paloaltonetworks.com/wp-content/uploads/2019/11/word-image-81-1024x585.png)_Figure 16: IP address check by the infected Windows host, right after HTTPS/SSL/TLS traffic over TCP port 449. Not inherently malicious, but this is part of a Trickbot infection._

A Trickbot infection currently generates HTTP traffic over TCP port 8082 this traffic sends information from the infected host like system information and passwords from the browser cache and email clients. This information is sent from the infected host to command and control servers used by Trickbot.

To review this traffic, use the following Wireshark filter:

**_http.request and tcp.port eq 8082_**

This reveals the following HTTP requests as seen in Figure 17:

*   170.238.117[.]187 port 8082 - **_170.238.117[.]187_** - POST  
    /ono19/BACHMANN-BTO-PC_W617601.AC3B679F4A22738281E6D7B0C5946  
    E42/81/
*   170.238.117[.]187 port 8082 - **_170.238.117[.]187_** - POST  
    /ono19/BACHMANN-BTO-PC_W617601.AC3B679F4A22738281E6D7B0C5946  
    E42/83/
*   170.238.117[.]187 port 8082 - **_170.238.117[.]187_** - POST  
    /ono19/BACHMANN-BTO-PC_W617601.AC3B679F4A22738281E6D7B0C5946  
    E42/81/
*   170.238.117[.]187 port 8082 - **_170.238.117[.]187:8082_** - POST  
    /ono19/BACHMANN-BTO-PC_W617601.AC3B679F4A22738281E6D7B0C5946  
    E42/81/
*   170.238.117[.]187 port 8082 - **_170.238.117[.]187:8082_** - POST  
    /ono19/BACHMANN-BTO-PC_W617601.AC3B679F4A22738281E6D7B0C5946  
    E42/90
*   170.238.117[.]187 port 8082 - **_170.238.117[.]187:8082_** - POST  
    /ono19/BACHMANN-BTO-PC_W617601.AC3B679F4A22738281E6D7B0C5946  
    E42/90

_![](https://unit42.paloaltonetworks.com/wp-content/uploads/2019/11/word-image-82-1024x362.png)Figure 17: HTTP traffic over TCP port 8082 caused by Trickbot._

HTTP POST requests ending in 81 send cached password data from web browsers, email clients, and other applications. HTTP POST requests ending in 83 send form data submitted by applications like web browsers. We can find system information sent through HTTP POST requests ending in 90. Follow the TCP or HTTP streams for any of these HTTP POST requests to review data stolen by this infection.

_![](https://unit42.paloaltonetworks.com/wp-content/uploads/2019/11/word-image-83-1024x804.png)Figure 18: Login credentials stolen by Trickbot from the Chrome web browser. This data was sent by the Trickbot-infected host using HTTP traffic over TCP port 8082._

_![](https://unit42.paloaltonetworks.com/wp-content/uploads/2019/11/word-image-84-1024x806.png)Figure 19: System data sent by a Trickbot-infected host using HTTP traffic over TCP port 8082. It starts with a list of running processes._

_![](https://unit42.paloaltonetworks.com/wp-content/uploads/2019/11/word-image-85-1024x802.png)Figure 20: More system data sent by a Trickbot-infected host using HTTP traffic over TCP port 8082. This is later from the same HTTP stream that started in Figure 19._

Trickbot sends more Windows executable files over HTTP GET requests ending in **_.png_**. These follow-up Trickbot executables are used to infect a vulnerable domain controller (DC) when the infected Windows host is a client in an Active Directory environment.

You can find these URLs in the pcap by using the following Wireshark filter:

 **_http.request and ip contains .png_**

_![](https://unit42.paloaltonetworks.com/wp-content/uploads/2019/11/word-image-86-1024x304.png)Figure 21: Filtering to find follow-up Trickbot EXE files sent using URLs ending with .png._

Follow the TCP or HTTP stream in each of the three requests as shown in Figure 21. You should see indicators of windows executable files similar to what we saw in Figure 9. However, in this case, the HTTP response headers identify the returned file as image/png even though it clearly is a Windows executable or DLL file.

_![](https://unit42.paloaltonetworks.com/wp-content/uploads/2019/11/word-image-87-1024x747.png)Figure 22: Windows executable sent through URL ending in .png._

You can export these files from Wireshark, confirm they are Windows executable files, and get the SHA256 file hashes as we covered earlier in this tutorial.

#### Trickbot Distributed Through Other Malware

Trickbot is frequently distributed through other malware. Trickbot is commonly seen as follow-up malware to Emotet infections, but we have also seen it as follow-up malware from IcedID and Ursnif infections

Since Emotet frequently distributes Trickbot, lets review an Emotet with Trickbot infection in September 2019 documented [here](https://www.malware-traffic-analysis.net/2019/09/25/index2.html). We already covered Emotet with Trickbot infections last year in [this Palo Alto Networks blog post](https://unit42.paloaltonetworks.com/unit42-malware-team-malspam-pushing-emotet-trickbot/), so this tutorial will focus on the Trickbot activity.

![](https://unit42.paloaltonetworks.com/wp-content/uploads/2019/11/word-image-88-1024x382.png)  
_Figure 23: Simplified flow chart for Emotet with Trickbot activity_.

Download the pcap [from this page](https://www.malware-traffic-analysis.net/2019/09/25/index.html). The pcap is contained in a password-protected zip archive named **_2019-09-25-Emotet-infection-with-Trickbot-in-AD-environment.pcap.zip_**. Extract the pcap from the zip archive using the password **_infected_** and open it in Wireshark. Use your **_basic_** filter to review the web-based infection traffic as shown in Figure 24.

![](https://unit42.paloaltonetworks.com/wp-content/uploads/2019/11/word-image-89-1024x588.png)_Figure 24: Filtering on web traffic in an Emotet+Trickbot infection._

Experienced analysts can usually identify the Emotet-generated traffic and the Trickbot-generated traffic. Post-infection Emotet activity consists HTTP traffic with encoded data returned by the server. This is distinctly different than post-infection Trickbot activity which generally relies on HTTPS/SSL/TLS traffic for command and control communications. Figure 25 points out the different infection traffic between Emotet and Trickbot for this specific infection.

![](https://unit42.paloaltonetworks.com/wp-content/uploads/2019/11/word-image-90-1024x586.png)_Figure 25: The differences in Emotet and Trickbot traffic._

This infection happened in an Active Directory environment with 10.9.25.102 as the infected Windows client and 10.9.25.9 as the DC. Later in the traffic, we see the DC exhibit signs of Trickbot infection as shown in Figure 26.

![](https://unit42.paloaltonetworks.com/wp-content/uploads/2019/11/word-image-91-1024x535.png)_Figure 26: Trickbot activity on the DC._

How did the infection move from client to DC? Trickbot uses a version of the [EternalBlue](https://www.wired.com/story/eternalblue-leaked-nsa-spy-tool-hacked-world/) exploit to move laterally using Microsoft’s SMB protocol. In this case, the infected Windows client sent information several times over TCP port 445 to the DC at 10.9.25.9, which then retrieved a Trickbot executable from **_185.98.87[.]185/wredneg2.png_**. Use the **_basic+_** filter to see the SYN segments for the traffic between the client at 10.9.25.102 and the DC at 10.9.25.9 right before the DC calls out to **_185.98.87[.]185_** as shown in Figure 27

![](https://unit42.paloaltonetworks.com/wp-content/uploads/2019/11/word-image-92-1024x703.png)_Figure 27: Finding traffic from the client at 10.9.25.102) to the DC at 10.9.25.9 (shown in grey) before the DC retrieved a Trickbot EXE from 196.98.87[.]185/wredneg2.png._

Follow one of the TCP streams, for example the line with a source as 10.9.25.102 over TCP port 49321 and destination as 10.9.35.9 over TCP port 445. This is highly unusual traffic for a client to send to a DC, so it is likely related to the EternalBlue exploit. See Figure 28 for an example of this traffic

_![](https://unit42.paloaltonetworks.com/wp-content/uploads/2019/11/word-image-93-1024x725.png)Figure 28: Example of the unusual traffic from a client to DC over TCP port 445, possibly related to an EternalBlue-based exploit._

Other than this unusual SMB traffic and the DC getting infected, any Trickbot-specific activity in this pcap is remarkably similar to our previous example.

Conclusion
----------

This tutorial provided tips for examining Windows infections with Trickbot malware by reviewing two pcaps from September 2019. More pcaps with recent examples of Trickbot activity can be found at [malware-traffic-analysis.net](https://www.malware-traffic-analysis.net/index.html).

For more help with Wireshark, see our previous tutorials:

*   [Customizing Wireshark - Changing Your Column Display](https://unit42.paloaltonetworks.com/unit42-customizing-wireshark-changing-column-display/)
*   [Using Wireshark - Display Filter Expressions](https://unit42.paloaltonetworks.com/using-wireshark-display-filter-expressions/)
*   [Using Wireshark: Identifying Hosts and Users](https://unit42.paloaltonetworks.com/using-wireshark-identifying-hosts-and-users/)
*   [Using Wireshark: Exporting Objects from a Pcap](https://unit42.paloaltonetworks.com/using-wireshark-exporting-objects-from-a-pcap/)

#### Get updates from  
Palo Alto  
Networks!

Sign up to receive the latest news, cyber threat intelligence and research from us