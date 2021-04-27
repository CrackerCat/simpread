> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [unit42.paloaltonetworks.com](https://unit42.paloaltonetworks.com/unit42-customizing-wireshark-changing-column-display/)

![](https://unit42.paloaltonetworks.com/wp-content/uploads/2019/11/tutorial-900x512-900x512.png)

This post is also available in: [日本語 (Japanese)](https://unit42.paloaltonetworks.jp/unit42-customizing-wireshark-changing-column-display/)

[Wireshark](https://www.wireshark.org/) is a free protocol analyzer that can record and display packet captures (pcaps) of network traffic. This tool is used by IT professionals to investigate a wide range of network issues. As a Threat Intelligence Analyst for Palo Alto Networks Unit 42, I often use Wireshark to review traffic generated from malware samples.  
What makes Wireshark so useful? It is very customizable. The default column display in Wireshark provides a wealth of information, but you should customize Wireshark to better meet your specific needs. This blog provides customization options helpful for security professionals investigating malicious network traffic.  
A pcap for this tutorial is available [here](https://www.malware-traffic-analysis.net/training/index.html). Keep in mind you must understand network traffic fundamentals to effectively use Wireshark. This tutorial uses version 2.6 of Wireshark and covers the following areas:

*   Web traffic and the default Wireshark column display
*   Hiding columns
*   Removing columns
*   Adding columns
*   Changing time to UTC
*   Custom columns

Web Traffic and the Default Wireshark Column Display  
Malware distribution frequently occurs through web traffic, and we also see this channel used for data exfiltration and command and control activity. Wireshark's default column is not ideal when investigating such malware-based infection traffic. However, Wireshark can be customized to provide a better view of the activity.  
![](https://unit42.paloaltonetworks.com/wp-content/uploads/2018/08/wireshark_1.png)

_Figure 1: Viewing a pcap using Wireshark's default column display._

Wireshark's default columns are:

*   **No**. -Frame number from the beginning of the pcap. The first frame is always 1.
*   **Time** - Seconds broken down to the nanosecond from the first frame of the pcap. The first frame is always 0.000000.
*   **Source** - Source address, commonly an IPv4, IPv6, or Ethernet address.
*   **Destination** - Destination address, commonly an IPv4, IPv6, or Ethernet address.
*   **Protocol** - Protocol used in the Ethernet frame, IP packet, or TCP segment (ARP, DNS, TCP, HTTP, etc.).
*   **Length** - Length of the frame in bytes.

In my day-to-day work, I require the following columns in my Wireshark display:

*   Date & time in UTC
*   Source IP and source port
*   Destination IP and destination port
*   HTTP host
*   HTTPS server
*   Info

How can we reach this state? First, we hide or remove the columns we do not want.

Hiding Columns  
We can easily hide columns in case we need them later. Right-click on any of the column headers to bring up the column header menu. Then left-click any of the listed columns to uncheck them. Figure 2 shows the **No.**, **Protocol**, and **Length** columns unchecked and hidden.  
![](https://unit42.paloaltonetworks.com/wp-content/uploads/2018/08/wireshark_2.png)

_Figure 2: Before and after shots of the column header menu when hiding columns._

Removing Columns  
Because I never use the **No.**, **Protocol**, or **Length** columns, I completely remove them. To remove columns, right-click on the column headers you want to remove. Then select "Remove this Column..." from the column header menu.  
![](https://unit42.paloaltonetworks.com/wp-content/uploads/2018/08/wireshark_3.png)

_Figure 3: Before and after shots of the column header menu when removing columns._

At this point, whether hidden or removed, the only visible columns are Time, Source, Destination, and Info.

Adding Columns  
To add columns in Wireshark, use the Column Preferences menu. Right-click on any of the column headers, then select "Column Preferences..."  
![](https://unit42.paloaltonetworks.com/wp-content/uploads/2018/08/wireshark_4.png)

_Figure 4: Getting to the Column Preferences menu by right-clicking on the column headers._

The Column Preferences menu lists all columns, viewed or hidden. Near the bottom left side of the Column Preferences menu are two buttons. One has a plus sign to add columns. The other has a minus sign to remove columns. Left-click on the plus sign. An entry titled "New Column" should appear at the bottom of the column list.  
![](https://unit42.paloaltonetworks.com/wp-content/uploads/2018/08/wireshark_5.png)

_Figure 5: Adding a new column in the Column Preferences menu._

Double-click on the "New Column" and rename it as "Source Port." The column type for any new columns always shows "Number." Double-click on "Number" to bring up a menu, then scroll to "Src port (unresolved)" and select that for the column type.  
![](https://unit42.paloaltonetworks.com/wp-content/uploads/2018/08/wireshark_6.png)

_Figure 6: Changing the column title._

![](https://unit42.paloaltonetworks.com/wp-content/uploads/2018/08/wireshark_7.png)

_Figure 7: Changing the column type._

Our new column is now named "Source Port" with a column type of "Src port (unresolved)." Left-click on that entry and drag it to a position immediately after the source address.

![](https://unit42.paloaltonetworks.com/wp-content/uploads/2018/08/wireshark_8.png)

_Figure 8: Changing the column position._

After the source port has been, add another column titled "Destination Port" with the column type "Dest port (unresolved)."  
![](https://unit42.paloaltonetworks.com/wp-content/uploads/2018/08/wireshark_9.png)

_Figure 9: Adding another column for Destination Port._

Like we did with the source port column, drag the destination port to place it immediately after the Destination address.  When you finish, your columns should appear as shown in Figure 10.

![](https://unit42.paloaltonetworks.com/wp-content/uploads/2018/08/wireshark_10.png)

_Figure 10: Final setup in the Column Preferences window._

After adding the source and destination port columns, click the "OK" button to apply the changes. These new columns are automatically aligned to the right, so right-click on each column header to align them to the left, so they match the other columns.

![](https://unit42.paloaltonetworks.com/wp-content/uploads/2018/08/wireshark_11.png)

_Figure 11: Aligning column displays in Wireshark._

![](https://unit42.paloaltonetworks.com/wp-content/uploads/2018/08/wireshark_12.png)

_Figure 12: Column display after adding and aligning the source and destination ports._

In my day-to-day work, I often hide the source address and source port columns until I need them.

Changing Time to UTC  
To change the time display format, go the "View" menu, maneuver to "Time Display Format," and change the value from "Seconds Since Beginning of Capture" to "UTC Date and Time of Day." Use the same menu path to change the resolution from "Automatic" to "Seconds." Figure 13 shows the menu paths for these options.  
![](https://unit42.paloaltonetworks.com/wp-content/uploads/2018/08/wireshark_13.png)

_Figure 13: Changing the time display format to UTC date and time._

![](https://unit42.paloaltonetworks.com/wp-content/uploads/2018/08/wireshark_14.png)

_Figure 14: UTC date and time as seen in updated Wireshark column display._

Adding Custom Columns  
While we can add several different types of columns through the column preferences menu, we cannot add every conceivable value. Fortunately, Wireshark allows us to add custom columns based on almost any value found in the frame details window. This is how we add domain names used in HTTP and HTTPS traffic to our Wireshark column display.  
To quickly find domains used in HTTP traffic, use the Wireshark filter **http.request** and examine the frame details window.  
In the frame details window, expand the line titled "Hypertext Transfer Protocol" by left clicking on the arrow that looks like a greater than sign to make it point down. This reveals several additional lines. Scroll down to the line starting with "Host:" to see the HTTP host name. Left click on this line to select it. Right click on the line to bring up a menu. Near the top of this menu, select "Apply as Column." This should create a new column with the HTTP host name.  
![](https://unit42.paloaltonetworks.com/wp-content/uploads/2018/08/wireshark_15.png)

_Figure 15: Applying the HTTP host name as a column._

![](https://unit42.paloaltonetworks.com/wp-content/uploads/2018/08/wireshark_16.png)

_Figure 16: HTTP host names in the column display when filtering on **http.request**._

To find domains used in encrypted HTTPS traffic, use the Wireshark filter **ssl.handshake.type == 1** and examine the frame details window.  
In the frame details window, expand the line titled "Secure Sockets Layer." Then expand the line for the TLS Record Layer. Below that expand another line titled "Handshake Protocol: Client Hello."  
![](https://unit42.paloaltonetworks.com/wp-content/uploads/2018/08/wireshark_17.png)

_Figure 17: Filtering on SSL handshake type and working our way down._

Below the "Handshake Protocol: Client Hello" line, expand the line that starts with "Extension: server_name." Under that is "Server Name Indication extension" which contains several Server Name value types when expanded. Select the line that starts with "Server Name:" and apply it as a column. Figure 18 shows an example.

![](https://unit42.paloaltonetworks.com/wp-content/uploads/2018/08/wireshark_18.png)

_Figure 18: Applying the HTTPS server name as a column._

![](https://unit42.paloaltonetworks.com/wp-content/uploads/2018/08/wireshark_19.png)

_Figure 19: HTTP server names in the column display when filtering on **ssl.handshake.type == 1**._

With this customization, we can filter on **http.request or ssl.handshake.type == 1** as shown in Figure 20. This gives us a much better idea of web traffic in a pcap than using the default column display in Wireshark.

![](https://unit42.paloaltonetworks.com/wp-content/uploads/2018/08/wireshark_20.png)

_Figure 20: Filtering on **http.request or ssl.handshake.type == 1** in the pcap for this tutorial._

Summary  
This tutorial covered the following areas:

*   Web traffic and the default Wireshark column display
*   Hiding columns
*   Removing columns
*   Adding columns
*   Changing time to UTC
*   Custom columns

Wireshark customization is helpful for security professionals investigating suspicious network traffic. It can be extremely useful when reviewing web traffic to determine an infection chain. Using the methods in this tutorial, we can configure Wireshark's column display to better fit our investigative workflow.

#### Get updates from  
Palo Alto  
Networks!

Sign up to receive the latest news, cyber threat intelligence and research from us