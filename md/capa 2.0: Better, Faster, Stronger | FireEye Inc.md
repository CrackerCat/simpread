> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [www.fireeye.com](https://www.fireeye.com/blog/threat-research/2021/07/capa-2-better-stronger-faster.html)

> We are excited to announce version 2.0 of our open-source tool, capa, which supports both malware tri......

We are excited to announce version 2.0 of our open-source tool called capa. capa automatically identifies capabilities in programs using an extensible rule set. The tool supports both malware triage and deep dive reverse engineering. If you haven’t heard of capa before, or need a refresher, check out our [first blog post](https://www.fireeye.com/blog/threat-research/2020/07/capa-automatically-identify-malware-capabilities.html). You can download capa 2.0 standalone binaries from the project’s [release page](https://github.com/fireeye/capa/releases) and checkout the source code on [GitHub](https://github.com/fireeye/capa).

capa 2.0 enables anyone to contribute rules more easily, which makes the existing ecosystem even more vibrant. This blog post details the following major improvements included in capa 2.0:

*   New features and enhancements for the [capa explorer](https://github.com/fireeye/capa/tree/master/capa/ida/plugin) IDA Pro plugin, allowing you to interactively explore capabilities and write new rules without switching windows
*   More concise and relevant results via identification of library functions using FLIRT and the release of accompanying open-source FLIRT signatures
*   Hundreds of new rules describing additional malware capabilities, bringing the collection up to 579 total rules, with more than half associated with ATT&CK techniques
*   Migration to Python 3, to make it easier to integrate capa with other projects

#### capa explorer and Rule Generator

capa explorer is an IDAPython plugin that shows capa results directly within IDA Pro. The version 2.0 release includes many additions and improvements to the plugin, but we'd like to highlight the most exciting addition: capa explorer now helps you write new capa rules directly in IDA Pro!

Since we spend most of our time in reverse engineering tools such as IDA Pro analyzing malware, we decided to add a capa rule generator. Figure 1 shows the rule generator interface.

![](https://www.fireeye.com/content/dam/fireeye-www/blog/images/capa-2/fig1.png)  
Figure 1: capa explorer rule generator interface

Once you’ve installed capa explorer using the [Getting Started](https://github.com/fireeye/capa/tree/master/capa/ida/plugin#getting-started) guide, open the plugin by navigating to _Edit_ > _Plugins_ > _FLARE capa explorer_. You can start using the rule generator by selecting the _Rule Generator_ tab at the top of the capa explorer pane. From here, navigate your IDA Pro Disassembly view to the function containing a technique you'd like to capture and click the _Analyze_ button. The rule generator will parse, format, and display all the capa features that it finds in your function. You can write your rule using the rule generator's three main panes: _Features_, _Preview_, and _Editor_. Your first step is to add features from the _Features_ pane.

The _Features_ pane is a tree view containing all the capa features extracted from your function. You can filter for specific features using the search bar at the top of the pane. Then, you can add features by double-clicking them. Figure 2 shows this in action.

![](https://www.fireeye.com/content/dam/fireeye-www/blog/images/capa-2/fig2.gif)  
Figure 2: capa explorer feature selection

As you add features from the _Features_ pane, the rule generator automatically formats and adds them to the _Preview_ and _Editor_ panes. The _Preview_ and _Editor_ panes help you finesse the features that you've added and allow you to modify other information like the rule's metadata.

The _Editor_ pane is an interactive tree view that displays the statement and feature hierarchy that forms your rule. You can reorder nodes using drag-and-drop and edit nodes via right-click context menus. To help humans understand the rule logic, you can add descriptions and comments to features by typing in the _Description_ and _Comment_ columns. The rule generator automatically formats any changes that you make in the _Editor_ pane and adds them to the _Preview_ pane. Figure 3 shows how to manipulate a rule using the _Editor_ pane.

![](https://www.fireeye.com/content/dam/fireeye-www/blog/images/capa-2/fig3.gif)  
Figure 3: capa explorer editor pane

The _Preview_ pane is an editable textbox containing the final rule text. You can edit any of the text displayed. The rule generator automatically formats any changes that you make in the _Preview_ pane and adds them to the _Editor_ pane. Figure 4 shows how to edit a rule directly in the _Preview_ pane.

![](https://www.fireeye.com/content/dam/fireeye-www/blog/images/capa-2/fig4.gif)  
Figure 4: capa explorer preview pane

As you make edits the rule generator lints your rule and notifies you of any errors using messages displayed underneath the _Preview_ pane. Once you've finished writing your rule you can save it to your capa rules directory by clicking the _Save_ button. The rule generator saves exactly what is displayed in the _Preview_ pane. It’s that simple!

We’ve found that using the capa explorer rule generator significantly reduces the amount of time spent writing new capa rules. This tool not only automates most of the rule writing process but also eliminates the need to context switch between IDA Pro and your favorite text editor allowing you to codify your malware knowledge while it’s fresh in your mind.

To learn more about capa explorer and the rule generator check out the [README](https://github.com/fireeye/capa/tree/master/capa/ida/plugin).

#### Library Function Identification Using FLIRT

As we wrote hundreds of capa rules and inspected thousands of capa results, we recognized that the tool sometimes shows distracting results due to embedded library code. We believe that capa needs to focus its attention on the programmer’s logic and ignore supporting library code. For example, highly optimized C/C++ runtime routines and open-source library code enable a programmer to quickly build a product but are not the _product itself_. Therefore, capa results should reflect the programmer’s intent for the program rather than a categorization of every byte in the program.

Compare the capa v1.6 results in Figure 5 versus capa v2.0 results in Figure 6. capa v2.0 identifies and skips almost 200 library functions and produces more relevant results.

![](https://www.fireeye.com/content/dam/fireeye-www/blog/images/capa-2/fig5.png)  
Figure 5: capa v1.6 results without library code recognition

![](https://www.fireeye.com/content/dam/fireeye-www/blog/images/capa-2/fig6.png)  
Figure 6: capa v2.0 results ignoring library code functions

So, we searched for a way to differentiate a programmer’s code from library code.

After experimenting with a few strategies, we landed upon the Fast Library Identification and Recognition Technology (FLIRT) [developed by Hex-Rays](https://hex-rays.com/products/ida/tech/flirt/in_depth/). Notably, this technique has remained stable and effective since 1996, is fast, requires very limited code analysis, and enjoys a wide community in the IDA Pro userbase. We figured out how IDA Pro matches FLIRT signatures and [re-implemented a matching engine in Rust](https://crates.io/crates/lancelot-flirt) with [Python bindings](https://pypi.org/project/python-flirt/). Then, we built an [open-source signature set](https://github.com/fireeye/siglib/) that covers many of the library routines encountered in modern malware. Finally, we updated capa to use the new signatures to guide its analysis.

capa uses these signatures to differentiate library code from a programmer’s code. While capa can extract and match against the names of embedded library functions, it will skip finding capabilities and behaviors within the library code. This way, capa results better reflect the logic written by a programmer.

Furthermore, library function identification drastically improves capa runtime performance: since capa skips processing of library functions, it can avoid the costly rule matching steps across a substantial percentage of real-world functions. Across our testbed of 206 samples, 28% of the 186,000 total functions are recognized as library code by our function signatures. As our implementation can recognize around 100,000 functions/sec, library function identification overhead is negligible and capa is approximately 25% faster than in 2020!

Finally, we introduced a new feature class that rule authors can use to match recognized library functions: function-name. This feature matches at the file-level scope. We’ve already started using this new capability to recognize specific implementations of cryptography routines, such as AES provided by [Crypto++](https://www.cryptopp.com/), as shown in the example rule in Figure 7.

![](https://www.fireeye.com/content/dam/fireeye-www/blog/images/capa-2/fig7.png)  
Figure 7: Example rule using function-name to recognize AES via Crypto++

As we developed rules for interesting behaviors, we learned a lot about where uncommon techniques are used legitimately. For example, as malware analysts, we most commonly see the cpuid instruction alongside anti-analysis checks, such as in VM detection routines. Therefore, we naively crafted rules to flag this instruction. But, when we tested it against our testbed, the rule matched most modern programs because this instruction is often legitimately used in high-optimized routines, such as memcpy, to opt-in to newer CPU features. In hindsight, this is obvious, but at the time it was a little surprising to see cpuid in around 15% of all executables. With the new FLIRT support, capa recognizes the optimized memcpy routine embedded by Visual Studio and won’t flag the embedded cpuid instruction, as its not part of the programmer’s code.

When a user upgrades to capa 2.0, they’ll see that the tool runs faster and provides more precise results.

#### Signature Generation

To provide the benefits of [python-flirt](https://pypi.org/project/python-flirt/) to all users (especially those without an IDA Pro license) we have spent significant time to create a comprehensive FLIRT signature set for the common malware analysis use-case. The signatures come included with capa and are also available at [our GitHub](https://github.com/fireeye/siglib) under the Apache 2.0 license. We believe that other projects can benefit greatly from this. For example, we expect the performance of [FLOSS](https://github.com/fireeye/flare-floss) to improve once we’ve incorporated library function identification. Moreover, you can use our signatures with IDA Pro to recognize more library code.

Our initial signatures include:

*   From Microsoft Visual Studio (VS), for all major versions from VS6 to VS2019:
    *   C and C++ run-time libraries
    *   Active Template Library (ATL) and Microsoft Foundation Class (MFC) libraries
*   The following open-source projects as compiled with VS2015, VS2017, and VS2019:
    *   CryptoPP
    *   curl
    *   Microsoft Detours
    *   Mbed TLS (previously PolarSSL)
    *   OpenSSL
    *   zlib

Identifying and collecting the relevant library and object files took a lot of work. For the older VS versions this was done manually. For newer VS versions and the respective open-source projects we were able to automate the process using [vcpgk](https://vcpkg.io/) and Docker.

We then used the IDA Pro FLAIR utilities to convert gigabytes of executable code into pattern files and then into signatures. This process required extensive research and much trial and error. For instance, we spent two weeks testing and exploring the various FLAIR options to understand the best combination. We appreciate Hex-Rays for providing high-quality signatures for IDA Pro and thank them for sharing their research and tools with the community.

To learn more about the pattern and signature file generation check out the [siglib](https://github.com/fireeye/siglib) repository. The FLAIR utilities are available in the protected download area on [Hex-Rays’ website](https://www.hex-rays.com/products/ida/support/download/).

#### Rule Updates

Since the initial release, the community has more than doubled the total capa rule count from 260 to over [570 capability detection rules](https://github.com/fireeye/capa-rules)! This means that capa recognizes many more techniques seen in real-world malware, certainly saving analysts time as they reverse engineer programs. And to reiterate, we’ve surfed a wave of support as almost 30 colleagues from a dozen organizations have volunteered their experience to develop these rules. Thank you!

Figure 8 provides a high-level overview of capabilities capa currently captures, including:

*   **Host Interaction** describes program functionality to interact with the file system, processes, and the registry
*   **Anti-Analysis** describes packers, Anti-VM, Anti-Debugging, and other related techniques
*   **Collection** describes functionality used to steal data such as credentials or credit card information
*   **Data Manipulation** describes capabilities to encrypt, decrypt, and hash data
*   **Communication** describes data transfer techniques such as HTTP, DNS, and TCP

![](https://www.fireeye.com/content/dam/fireeye-www/blog/images/capa-2/fig8.png)  
Figure 8: Overview of capa rule categories

More than half of capa’s rules are associated with a [MITRE ATT&CK](https://attack.mitre.org/) technique including all techniques introduced in ATT&CK [version 9](https://medium.com/mitre-attack/attack-april-2021-release-39accaf23c81) that lie within capa’s scope. Moreover, almost half of the capa rules are currently associated with a [Malware Behavior Catalog](https://github.com/MBCProject/mbc-markdown) (MBC) identifier.

For more than 70% of capa rules we have collected associated real-world binaries. Each binary implements interesting capabilities and exhibits noteworthy features. You can view the entire sample collection at our [capa test files GitHub page](https://github.com/fireeye/capa-testfiles). We rely heavily on these samples for developing and testing code enhancements and rule updates.

#### Python 3 Support

Finally, we’ve spent nearly three months migrating capa from Python 2.7 to Python 3. This involved working closely with [vivisect](https://github.com/vivisect/vivisect) and we would like to thank the team for their support. After extensive testing and a couple of releases supporting two Python versions, we’re excited that capa 2.0 and future versions will be Python 3 only.

#### Conclusion

Now that you’ve seen all the recent improvements to capa, we hope you’ll upgrade to the newest capa version right away! Thanks to library function identification capa will report faster and more relevant results. Hundreds of new rules capture the most interesting malware functionality while the improved capa explorer plugin helps you to focus your analysis and codify your malware knowledge while it’s fresh.

Standalone binaries for Windows, Mac, and Linux are available on the [capa Releases page](https://github.com/fireeye/capa/releases). To install capa from [PyPi](https://pypi.org/project/flare-capa/) use the command pip install flare-capa. The source code is available at our [capa GitHub page](https://github.com/fireeye/capa). The project page on GitHub contains detailed documentation, including thorough [installation](https://github.com/fireeye/capa/blob/master/doc/installation.md) instructions and a walkthrough of [capa explorer](https://github.com/fireeye/capa/blob/master/capa/ida/plugin/README.md). Please use [GitHub](https://github.com/fireeye/capa/issues) to ask questions, discuss ideas, and submit issues.

We highly encourage you to contribute to capa’s rule corpus. The improved IDA Pro plugin makes it easier than ever before. If you have any issues or ideas related to rules, please let us know on the [GitHub repository](https://github.com/fireeye/capa-rules). Remember, when you share a rule with the community, you scale your impact across hundreds of reverse engineers in dozens of organizations.