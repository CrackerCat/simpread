> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [pfeifferszilard.hu](https://pfeifferszilard.hu/2021/12/27/cryptolyzer-a-comprehensive-cryptographic-settings-analyzer.html)

> CryptoLyzer is a multiprotocol cryptographic settings analyzer with SSL/TLS, SSH, and HTTP header ana......

**CryptoLyzer is a multiprotocol cryptographic settings analyzer with SSL/TLS, SSH, and HTTP header analysis ability. The main purpose of the tool is to tell you what kind of cryptographic related settings are enabled on a client or server.**

If you are not interested in the principles behind the project, but the practice, feel free to skip the next section and jump to the Practice section.

Rationale[Permalink](#rationale "Permalink")
--------------------------------------------

There are many notable open-source projects ([SSLyze](https://github.com/nabla-c0d3/sslyze), [CipherScan](https://github.com/mozilla/cipherscan), [testssl.sh](https://testssl.sh/), [tls-scan](https://github.com/prbinu/tls-scan), …) and several SaaS solutions ([CryptCheck](https://tls.imirhil.fr/), [CypherCraft](https://www.cyphercraft.io/), [Hardenize](https://hardenize.com/), [ImmuniWeb](https://www.immuniweb.com/ssl/), [Mozilla Observatory](https://observatory.mozilla.org/), [SSL Labs](https://www.ssllabs.com/), …) to do a security setting analysis, especially when we are talking about TLS, which is the most common and popular cryptographic protocol. However, most of these tools heavily depend on one or more versions of one or more cryptographic protocol libraries, like [GnuTLS](https://www.gnutls.org/), [OpenSSL](https://www.openssl.org/), or [wolfSSL](https://www.wolfssl.com/). But why is this such a problem?

The minor problem is the dependency easily stucks them in [SSL](https://en.wikipedia.org/wiki/Transport_Layer_Security#SSL_1.0,_2.0,_and_3.0)/[TLS](https://en.wikipedia.org/wiki/Transport_Layer_Security)/[DTLS](https://en.wikipedia.org/wiki/Datagram_Transport_Layer_Security) as other cryptographic protocols (eg: [IPSec VPN](https://en.wikipedia.org/wiki/IPsec), [Kerberos](https://en.wikipedia.org/wiki/Kerberos_(protocol)), [OpenVPN](https://en.wikipedia.org/wiki/OpenVPN), [SSH](https://en.wikipedia.org/wiki/Secure_Shell_Protocol), …) can not be implemented directly by these libraries. Supporting them by the analyzer application needs extra effort. Anyway, most of the analysis of the cryptographic setting does not require any cryptography because before the parties could agree on the cryptographic algorithms they use plain text.

The major problem is the fact that analysis should test special and corner cases of the protocol that are intentionally triggered. It is hard to do that with a cryptographic protocol library, which was designed for production not for penetration testing or settings analysis. During an analysis, connections are tried to be established with hardly supported, experimental, obsoleted, or even deprecated mechanisms or algorithms to identify which ones are supported by the given client or server implementation. These mechanisms and algorithms may or may not be supported by the latest or any version of any cryptographic protocol implementations.

That is why most of the existing tools require special build(s) of the dependent library where all the protocol versions and algorithms of the good old days are reenabled to make the chance to set up these libraries to offer them as clients or as servers. But what if we want to test an algorithm or a mechanism that has never been implemented by the dependent cryptographic library?

It is not just a theory. A [special fork](https://github.com/PeterMosmans/openssl) of OpenSSL, maintained by Pluralsight author [Peter Mosmans](https://www.pluralsight.com/authors/peter-mosmans), aims to have as many ciphers as possible. This fork is used and recommended by Mozilla Cipherscan, however, it can offer [less than two hundred](https://github.com/PeterMosmans/openssl/blob/1.0.2-chacha/README.md) cipher suites, but there are more than three hundred in the different [RFC](https://en.wikipedia.org/wiki/Request_for_Comments)s according to [Cipher Suite Info](https://ciphersuite.info/cs/?software=all&singlepage=true). The majority of them are weak or insecure, which makes it particularly important to be part of the analysis. In addition, it is also true that there are cipher suites that are not on the Cipher Suite Info list, for instance, Russian standard ([GOST](https://en.wikipedia.org/wiki/GOST)) cipher suites. These are rarely used cipher suites, but there is an [OpenSSL engine](https://github.com/gost-engine/engine) that implements them, so they should be checked.

The situation is similar in the case of other parts of the TLS protocol, like extensions. It is unlikely that a cryptographic library will support each extension, but some of them are possibly implemented by some of the libraries. For instance, [multiple certificate status request extension](https://datatracker.ietf.org/doc/html/rfc6961) and Diffie-Hellman key exchange in TLS 1.3 are not implemented by OpenSSL 1.x but implemented by GnuTLS and WolfSSL. Another example can be the [X.509 certificate-based authentication in SSH](https://datatracker.ietf.org/doc/html/rfc6187) protocol, which is supported by [Tectia SSH](https://www.ssh.com/manuals/server-admin/60/serverauth-cert.html) and has an [open-source implementation](http://roumenpetrov.info/openssh/) but is not supported by the default OpenSSH. Even though they are not so common a comprehensive settings analyzer tool should run checks against them.

Summing up the main reason for establishing a greenfield project instead of contributing to existing one is the opinion, that an analysis is mostly testing when we trigger special and corner cases, hardly supported, experimental, obsoleted, or even deprecated mechanisms and algorithms, so a comprehensive analyzer should be implemented independently from current cryptographic protocol implementations as much as possible.

Goals[Permalink](#goals "Permalink")
------------------------------------

CrypytoLyzer focuses on comprehensiveness, multi-protocol ability, and library independence. To do so cryptographic protocols clients are implemented focusing only on the necessary parts (usually the handshake). This kind of client can check a server against rarely or privately used, deprecated, or completely insecure algorithms and mechanisms.

*   Zero Trust
    *   should not require any privilege during the analysis
    *   Should not require any credentials (except it is strictly necessary)
    *   should not provide any information for 3rd party
*   Completeness
    *   should be able to support as many cryptographic protocols (DTLS, IPSec, OpenVPN, TLS, SSL, SSH, …) as possible
    *   should be able to handle special/corner cases of the cryptographic protocols
*   Usability
    *   should work both in public and private networks
    *   should be able to give a human-readable summary ([Markdown](https://en.wikipedia.org/wiki/Markdown))
    *   should be able to give machine-readable detailed output ([JSON](https://en.wikipedia.org/wiki/JSON))
    *   should be customizable to meet special needs (Python [library](https://en.wikipedia.org/wiki/Library_(computing)))

The existing solutions are focusing on only one cryptographic protocol (TLS or SSH) despite the fact, that all the cryptographic protocols have the same building blocks (authentication, key exchange, symmetric ciphers, integrity), so they can be analyzed in the same (or almost the same) way, only the information which the analysis is based on can be acquired differently.

The goals above are just goals, not the current state of the development, particularly since the actual version number is 0.8.0 at the time of writing. The supported cryptographic protocol families, for now, are the SSL/TLS and SSH. You can read comparative analysis with either open source or proprietary (SaaS) solutions below. I wanted to stay as objective as an author can with the competitors, but forgive me if I have not been managed to.

Strengths[Permalink](#strengths "Permalink")
--------------------------------------------

### Transport Layer Security (TLS)[Permalink](#transport-layer-security-tls "Permalink")

#### Specialties[Permalink](#specialties "Permalink")

Cryptolyzer supports both the early and deprecated [Secure Socket Layer (SSL) 2.0](https://tools.ietf.org/html/draft-hickman-netscape-ssl-00) and each [Transport Layer Security](https://en.wikipedia.org/wiki/Transport_Layer_Security) version up to the [TLS 1.3](https://tools.ietf.org/html/rfc8446) version including draft versions. Some features although be checked hardly or difficultly using the most popular SSL/TLS implementations (eg: GnuTLS, [LibreSSL](https://www.libressl.org/), OpenSSL, wolfSSL, …) or just not in the scope of them. The checks are the specialties of Cryptolyzer such as supporting each cipher suite discussed on [ciphersuite.info](https://ciphersuite.info/) (and more) including the Russian governmental standard (GOST) cipher suites, Diffie-Hellman parameter checks, JA3 tag generation, …

<table><thead><tr><th>&nbsp;</th><th><strong>Crypto Lyzer</strong></th><th><a href="https://www.ssllabs.com/ssltest/">Qualys SSL Labs</a></th><th><a href="https://www.immuniweb.com/ssl/">Immuni Web</a></th><th><a href="https://www.hardenize.com/">Hard enize</a></th><th><a href="https://observatory.mozilla.org/">Moz. Obs.</a></th><th><a href="https://testssl.sh/">testssl .sh</a></th><th><a href="https://github.com/nabla-c0d3/sslyze">SSL-yze</a></th><th>&nbsp;</th></tr></thead><tbody><tr><td>TLS 1.3 Draft Versions</td><td>✅</td><td>❌</td><td>❌</td><td>✅</td><td>❌</td><td>❌</td><td>✅</td><td>❌</td></tr><tr><td><a href="https://datatracker.ietf.org/doc/html/rfc8446#section-4.2.8.1">TLS 1.3 DH Parameters</a></td><td>✅</td><td>❌</td><td>❌</td><td>❌</td><td>❌</td><td>❌</td><td>✅</td><td>❌</td></tr><tr><td>TLS <a href="https://en.wikipedia.org/wiki/GOST">GOST</a> Ciphers</td><td>✅</td><td>❌</td><td>❌</td><td>❌</td><td>❌</td><td>❌</td><td>❌</td><td>❌</td></tr><tr><td>TLS DH Param <a href="https://en.wikipedia.org/wiki/Safe_and_Sophie_Germain_primes">Prime Check</a></td><td>✅</td><td>✅</td><td>❌</td><td>❌</td><td>❌</td><td>❌</td><td>❌</td><td>❌</td></tr><tr><td>TLS <a href="https://security.stackexchange.com/questions/225209/what-is-ecdh-public-server-param-reuse">DHE Param Reuse</a> Check</td><td>✅</td><td>✅</td><td>❌</td><td>❌</td><td>❌</td><td>❌</td><td>❌</td><td>❌</td></tr><tr><td>TLS ECDH Param Reuse Check</td><td><strong>0.9.x</strong></td><td>✅</td><td>❌</td><td>❌</td><td>❌</td><td>❌</td><td>❌</td><td>❌</td></tr><tr><td>SSH DH <a href="https://en.wikipedia.org/wiki/Safe_and_Sophie_Germain_primes">Param Check</a></td><td>✅</td><td>❌</td><td>❌</td><td>❌</td><td>❌</td><td>❌</td><td>❌</td><td>❌</td></tr><tr><td><a href="https://engineering.salesforce.com/tls-fingerprinting-with-ja3-and-ja3s-247362855967">JA3 tag</a> Generation</td><td>✅</td><td>❌</td><td>❌</td><td>❌</td><td>❌</td><td>❌</td><td>❌</td><td>❌</td></tr><tr><td><a href="https://engineering.salesforce.com/tls-fingerprinting-with-ja3-and-ja3s-247362855967">JA3 tag</a> Decode</td><td>✅</td><td>❌</td><td>❌</td><td>❌</td><td>❌</td><td>❌</td><td>❌</td><td>❌</td></tr><tr><td>&nbsp;</td><td><strong>89%</strong></td><td><strong>33%</strong></td><td>0%</td><td>11%</td><td>0%</td><td>0%</td><td>22%</td><td>0%</td></tr></tbody></table>

#### Opportunistic TLS[Permalink](#opportunistic-tls "Permalink")

Several application-layer protocols have an extension to upgrade a plain text connection to an encrypted one (TLS) without using a separate port referred to as [opportunistic TLS](https://en.wikipedia.org/wiki/Opportunistic_TLS) (STARTTLS). This mechanism is supported by the analyzer in the case of most protocols.

<table><thead><tr><th>&nbsp;</th><th><strong>Crypto Lyzer</strong></th><th><a href="https://www.ssllabs.com/ssltest/">Qualys SSL Labs</a></th><th><a href="https://www.immuniweb.com/ssl/">Immuni Web</a></th><th><a href="https://www.hardenize.com/">Hard enize</a></th><th><a href="https://observatory.mozilla.org/">Moz. Obs.</a></th><th><a href="https://testssl.sh/">testssl .sh</a></th><th><a href="https://github.com/nabla-c0d3/sslyze">SSL-yze</a></th></tr></thead><tbody><tr><td><a href="https://en.wikipedia.org/wiki/File_Transfer_Protocol">FTP</a></td><td>✅</td><td>❌</td><td>❌</td><td>✅</td><td>❌</td><td>✅</td><td>✅</td></tr><tr><td><a href="https://en.wikipedia.org/wiki/Internet_Message_Access_Protocol">IMAP</a></td><td>✅</td><td>❌</td><td>❌</td><td>❌</td><td>❌</td><td>✅</td><td>✅</td></tr><tr><td><a href="https://en.wikipedia.org/wiki/Lightweight_Directory_Access_Protocol">LDAP</a></td><td>✅</td><td>❌</td><td>❌</td><td>❌</td><td>❌</td><td>❌</td><td>✅</td></tr><tr><td><a href="https://en.wikipedia.org/wiki/Local_Mail_Transfer_Protocol">LMTP</a></td><td>✅</td><td>❌</td><td>❌</td><td>❌</td><td>❌</td><td>✅</td><td>❌</td></tr><tr><td><a href="https://en.wikipedia.org/wiki/MySQL">MySQL</a></td><td><strong>0.9.x</strong></td><td>❌</td><td>❌</td><td>❌</td><td>❌</td><td>✅</td><td>❌</td></tr><tr><td><a href="https://en.wikipedia.org/wiki/Network_News_Transfer_Protocol">NNTP</a></td><td>✅</td><td>❌</td><td>❌</td><td>❌</td><td>❌</td><td>✅</td><td>❌</td></tr><tr><td><a href="https://en.wikipedia.org/wiki/Post_Office_Protocol">POP3</a></td><td>✅</td><td>❌</td><td>❌</td><td>❌</td><td>❌</td><td>✅</td><td>✅</td></tr><tr><td><a href="https://en.wikipedia.org/wiki/PostgreSQL">PostgreSQL</a></td><td>✅</td><td>❌</td><td>❌</td><td>❌</td><td>❌</td><td>✅</td><td>✅</td></tr><tr><td><a href="https://en.wikipedia.org/wiki/Remote_Desktop_Protocol">RDP</a></td><td>✅</td><td>❌</td><td>❌</td><td>❌</td><td>❌</td><td>❌</td><td>✅</td></tr><tr><td><a href="https://en.wikipedia.org/wiki/Sieve_(mail_filtering_language)">Sieve</a></td><td>✅</td><td>❌</td><td>❌</td><td>❌</td><td>❌</td><td>✅</td><td>❌</td></tr><tr><td><a href="https://en.wikipedia.org/wiki/Simple_Mail_Transfer_Protocol">SMTP</a></td><td>✅</td><td>❌</td><td>✅</td><td>✅</td><td>❌</td><td>✅</td><td>✅</td></tr><tr><td><a href="https://en.wikipedia.org/wiki/XMPP">XMPP (Jabber)</a></td><td>✅</td><td>❌</td><td>❌</td><td>❌</td><td>❌</td><td>✅</td><td>✅</td></tr><tr><td>&nbsp;</td><td><strong>92%</strong></td><td>0%</td><td>8%</td><td>17%</td><td>0%</td><td><strong>83%</strong></td><td>67%</td></tr></tbody></table>

#### Extensions[Permalink](#extensions "Permalink")

There are several extensions for the original TLS specifications which have both security and compatibility considerations. A high number of extension types can be analyzed by CryptoLyzer compared to most of the competitors.

<table><thead><tr><th>&nbsp;</th><th><strong>Crypto Lyzer</strong></th><th><a href="https://www.ssllabs.com/ssltest/">Qualys SSL Labs</a></th><th><a href="https://www.immuniweb.com/ssl/">Immuni Web</a></th><th><a href="https://www.hardenize.com/">Hard enize</a></th><th><a href="https://observatory.mozilla.org/">Moz. Obs.</a></th><th><a href="https://testssl.sh/">testssl .sh</a></th><th><a href="https://github.com/nabla-c0d3/sslyze">SSL-yze</a></th></tr></thead><tbody><tr><td><a href="https://datatracker.ietf.org/doc/html/rfc7507">Fallback SCSV</a></td><td>✅</td><td>✅</td><td>✅</td><td>❌</td><td>❌</td><td>✅</td><td>✅</td></tr><tr><td><a href="https://datatracker.ietf.org/doc/html/rfc5246#section-7.4.1.2">TLS Clock</a> Skew</td><td>✅</td><td>❌</td><td>❌</td><td>❌</td><td>❌</td><td>✅</td><td>✅</td></tr><tr><td><a href="https://datatracker.ietf.org/doc/html/rfc3749">Compression</a></td><td>✅</td><td>✅</td><td>✅</td><td>❌</td><td>❌</td><td>❌</td><td>✅</td></tr><tr><td><a href="https://datatracker.ietf.org/doc/html/rfc5746">Secure Renegotiation</a></td><td>✅</td><td>✅</td><td>✅</td><td>❌</td><td>❌</td><td>✅</td><td>✅</td></tr><tr><td>Insecure <a href="https://datatracker.ietf.org/doc/html/rfc5246#section-7.4.1.2">Renegotiation</a></td><td><strong>0.8.x</strong></td><td>✅</td><td>✅</td><td>❌</td><td>❌</td><td>✅</td><td>✅</td></tr><tr><td><a href="https://datatracker.ietf.org/doc/html/rfc5246#appendix-F.1.4">Session Caching</a></td><td>✅</td><td>✅</td><td>❌</td><td>❌</td><td>❌</td><td>✅</td><td>✅</td></tr><tr><td><a href="https://datatracker.ietf.org/doc/html/rfc5077">Session Ticketing</a></td><td>✅</td><td>✅</td><td>❌</td><td>❌</td><td>❌</td><td>✅</td><td>✅</td></tr><tr><td><a href="https://www.rfc-editor.org/rfc/rfc4492.html#section-5.1.1">Elliptic Curves</a></td><td>✅</td><td>✅</td><td>✅</td><td>❌</td><td>❌</td><td>✅</td><td>✅</td></tr><tr><td><a href="https://www.rfc-editor.org/rfc/rfc4492.html#section-5.1.1">EC Point Format</a></td><td>✅</td><td>❌</td><td>✅</td><td>❌</td><td>❌</td><td>❌</td><td>❌</td></tr><tr><td><a href="https://www.rfc-editor.org/rfc/rfc7301.html">App. Layer Proto. Negotiation</a></td><td>✅</td><td>✅</td><td>❌</td><td>❌</td><td>❌</td><td>✅</td><td>❌</td></tr><tr><td><a href="https://tools.ietf.org/id/draft-agl-tls-nextprotoneg-03.html">Next Protocol Negotiation</a></td><td>✅</td><td>✅</td><td>❌</td><td>❌</td><td>❌</td><td>✅</td><td>❌</td></tr><tr><td><a href="https://datatracker.ietf.org/doc/html/rfc7627">Extended Master Secret</a></td><td>✅</td><td>❌</td><td>❌</td><td>❌</td><td>❌</td><td>✅</td><td>❌</td></tr><tr><td><a href="https://www.rfc-editor.org/rfc/rfc7366.html">Encrypt-then-mac</a></td><td>✅</td><td>❌</td><td>❌</td><td>❌</td><td>❌</td><td>✅</td><td>❌</td></tr><tr><td><a href="https://www.rfc-editor.org/rfc/rfc8701.html">GREASE check</a></td><td><strong>0.9.x</strong></td><td>✅</td><td>❌</td><td>❌</td><td>❌</td><td>❌</td><td>❌</td></tr><tr><td><a href="https://security.stackexchange.com/questions/181692/long-handshake-intolerance-ssl-tls-command-line">Handshake Length Intolerance</a></td><td><strong>0.9.x</strong></td><td>✅</td><td>❌</td><td>❌</td><td>❌</td><td>❌</td><td>❌</td></tr><tr><td>&nbsp;</td><td><strong>80%</strong></td><td><strong>73%</strong></td><td>40%</td><td>0%</td><td>0%</td><td><strong>73%</strong></td><td>53%</td></tr></tbody></table>

### Secure Shell (SSH)[Permalink](#secure-shell-ssh "Permalink")

In addition to TLS, there are some other cryptographic protocols, such as SSH. The cryptographic protocols have the same building blocks (authentication, key exchange, symmetric ciphers, integrity), so they can be analyzed in the same manner and it can be done with the same tool using CryptoLyzer without compromise.

<table><thead><tr><th>&nbsp;</th><th><strong>Crypto Lyzer</strong></th><th><a href="https://observatory.mozilla.org/">Moz. Obs.</a></th><th><a href="https://sshcheck.operous.dev/">Operous SSH Check</a></th><th><a href="https://sshcheck.com/">Rebex SSH Check</a></th><th><a href="https://www.sshaudit.com/">SSH Conf. Auditor</a></th><th><a href="https://github.com/jtesta/ssh-audit">ssh-audit</a></th><th><a href="https://github.com/mozilla/ssh_scan">ssh-scan</a></th></tr></thead><tbody><tr><td>Algorithms</td><td>✅</td><td>✅</td><td>✅</td><td>✅</td><td>✅</td><td>✅</td><td>✅</td></tr><tr><td>Host Keys</td><td>✅</td><td>❌</td><td>✅</td><td>❌</td><td>✅</td><td>✅</td><td>✅</td></tr><tr><td>Host Certificates</td><td><strong>0.9.x</strong></td><td>❌</td><td>❌</td><td>❌</td><td>❌</td><td>❌</td><td>❌</td></tr><tr><td>X.509 Certificates</td><td><strong>0.10.x</strong></td><td>❌</td><td>❌</td><td>❌</td><td>❌</td><td>❌</td><td>❌</td></tr><tr><td>DH Param Check</td><td>✅</td><td>❌</td><td>❌</td><td>❌</td><td>❌</td><td>❌</td><td>❌</td></tr><tr><td>Banner Analysis</td><td>✅</td><td>❌</td><td>✅</td><td>❌</td><td>❌</td><td>❌</td><td>✅</td></tr><tr><td>&nbsp;</td><td><strong>67%</strong></td><td>17%</td><td>50%</td><td>17%</td><td>33%</td><td>33%</td><td><strong>50%</strong></td></tr></tbody></table>

#### Security[Permalink](#security "Permalink")

There are security mechanisms in the application layer too, not only in the transport layer. The HTTP protocol is a good example of that as there are encryption (HSTS), authentication (Expect-CT, Expect-Staple), content integrity (subresource integrity, content security policy) related headers which are parsed by CryptoLyzer in detail.

<table><thead><tr><th>&nbsp;</th><th><strong>Crypto Lyzer</strong></th><th><a href="https://www.ssllabs.com/ssltest/">Qualys SSL Labs</a></th><th><a href="https://www.immuniweb.com/ssl/">Immuni Web</a></th><th><a href="https://www.hardenize.com/">Hard enize</a></th><th><a href="https://observatory.mozilla.org/">Moz. Obs.</a></th><th><a href="https://testssl.sh/">testssl .sh</a></th><th><a href="https://github.com/nabla-c0d3/sslyze">SSL-yze</a></th></tr></thead><tbody><tr><td><a href="https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/Content-Security-Policy">Content Security Policy</a></td><td><strong>0.9.x</strong></td><td>❌</td><td>❌</td><td>❌</td><td>✅</td><td>❌</td><td>❌</td></tr><tr><td><a href="https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/Content-Security-Policy/report-to">CSP Report-To</a></td><td><strong>0.9.x</strong></td><td>❌</td><td>❌</td><td>❌</td><td>❌</td><td>❌</td><td>❌</td></tr><tr><td><a href="https://scotthelme.co.uk/a-new-security-header-expect-ct/">Expect Certificate Transparency</a></td><td>✅</td><td>❌</td><td>❌</td><td>✅</td><td>✅</td><td>❌</td><td>✅</td></tr><tr><td><a href="https://scotthelme.co.uk/designing-a-new-security-header-expect-staple/">Expect OCSP Staple</a></td><td>✅</td><td>❌</td><td>❌</td><td>❌</td><td>❌</td><td>❌</td><td>❌</td></tr><tr><td><a href="https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/Strict-Transport-Security">HTTP Strict Transport Security</a></td><td>✅</td><td>✅</td><td>✅</td><td>✅</td><td>✅</td><td>✅</td><td>✅</td></tr><tr><td><a href="https://developer.mozilla.org/en-US/docs/Web/HTTP/Public_Key_Pinning">HTTP Public Key Pinning</a></td><td><strong>0.10.x</strong></td><td>✅</td><td>❌</td><td>❌</td><td>✅</td><td>✅</td><td>❌</td></tr><tr><td><a href="https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/Referrer-Policy">Referrer Policy</a></td><td>✅</td><td>❌</td><td>❌</td><td>❌</td><td>✅</td><td>❌</td><td>❌</td></tr><tr><td><a href="https://developer.mozilla.org/en-US/docs/Web/Security/Subresource_Integrity">Subresource Integrity</a></td><td><strong>0.8.x</strong></td><td>❌</td><td>❌</td><td>✅</td><td>✅</td><td>❌</td><td>❌</td></tr><tr><td><a href="https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/X-Content-Type-Options">X Content Type Options</a></td><td>✅</td><td>❌</td><td>❌</td><td>✅</td><td>✅</td><td>❌</td><td>❌</td></tr><tr><td><a href="https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/X-Frame-Options">X Frame Options</a></td><td>✅</td><td>❌</td><td>❌</td><td>✅</td><td>✅</td><td>❌</td><td>❌</td></tr><tr><td><a href="https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/X-XSS-Protection">X XSS Protection</a></td><td><strong>0.8.x</strong></td><td>❌</td><td>❌</td><td>✅</td><td>✅</td><td>❌</td><td>❌</td></tr><tr><td>&nbsp;</td><td><strong>55%</strong></td><td>18%</td><td>9%</td><td>55%</td><td><strong>82%</strong></td><td>18%</td><td>18%</td></tr></tbody></table>

#### Generic[Permalink](#generic "Permalink")

Several HTTP headers are not closely related to security. Although, some among them, especially caching headers (Age, Cache-Control, Date, ETag, Expires, Last-Modified), can have significance when obsolescence of security-related information (eg: CRL) is indicated by caching headers.

<table><thead><tr><th>&nbsp;</th><th><strong>Crypto Lyzer</strong></th><th><a href="https://www.ssllabs.com/ssltest/">Qualys SSL Labs</a></th><th><a href="https://www.immuniweb.com/ssl/">Immuni Web</a></th><th><a href="https://www.hardenize.com/">Hard enize</a></th><th><a href="https://observatory.mozilla.org/">Moz. Obs.</a></th><th><a href="https://testssl.sh/">testssl .sh</a></th><th><a href="https://github.com/nabla-c0d3/sslyze">SSL-yze</a></th></tr></thead><tbody><tr><td><a href="https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/Content-Type">Content-Type</a></td><td>✅</td><td>❌</td><td>❌</td><td>❌</td><td>✅</td><td>❌</td><td>❌</td></tr><tr><td><a href="https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/Server">Server</a></td><td>✅</td><td>❌</td><td>❌</td><td>❌</td><td>✅</td><td>✅</td><td>❌</td></tr><tr><td>Application banner</td><td><strong>0.9.x</strong></td><td>❌</td><td>❌</td><td>❌</td><td>❌</td><td>✅</td><td>❌</td></tr><tr><td><a href="https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/Age">Age</a></td><td>✅</td><td>❌</td><td>❌</td><td>❌</td><td>❌</td><td>❌</td><td>❌</td></tr><tr><td><a href="https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/Cache-Control">Cache-Control</a></td><td>✅</td><td>❌</td><td>❌</td><td>❌</td><td>✅</td><td>❌</td><td>❌</td></tr><tr><td><a href="https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/Date">Date</a></td><td>✅</td><td>❌</td><td>❌</td><td>❌</td><td>✅</td><td>✅</td><td>❌</td></tr><tr><td><a href="https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/ETag">ETag</a></td><td>✅</td><td>❌</td><td>❌</td><td>❌</td><td>✅</td><td>❌</td><td>❌</td></tr><tr><td><a href="https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/Expires">Expires</a></td><td>✅</td><td>❌</td><td>❌</td><td>❌</td><td>✅</td><td>❌</td><td>❌</td></tr><tr><td><a href="https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/Last-Modified">Last-Modified</a></td><td>✅</td><td>❌</td><td>❌</td><td>❌</td><td>✅</td><td>❌</td><td>❌</td></tr><tr><td><a href="https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/Pragma">Pragma</a></td><td>✅</td><td>❌</td><td>❌</td><td>❌</td><td>✅</td><td>❌</td><td>❌</td></tr><tr><td>Proxy banner</td><td><strong>0.9.x</strong></td><td>❌</td><td>❌</td><td>❌</td><td>❌</td><td>✅</td><td>❌</td></tr><tr><td><a href="https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/Set-Cookie">Set-Cookie</a></td><td><strong>0.10.x</strong></td><td>❌</td><td>❌</td><td>✅</td><td>✅</td><td>✅</td><td>❌</td></tr><tr><td>&nbsp;</td><td><strong>75%</strong></td><td>0%</td><td>0</td><td>8%</td><td><strong>75%</strong></td><td>42%</td><td>0%</td></tr></tbody></table>

Weaknesses[Permalink](#weaknesses "Permalink")
----------------------------------------------

### Transport Layer Security (TLS)[Permalink](#transport-layer-security-tls-1 "Permalink")

#### X.509 Extensions[Permalink](#x509-extensions "Permalink")

There are some X.509 related mechanisms in TLS that can have a serious impact on security in particular the revocation checking which is a [weak point of the public key infrastructure](https://coroner.medium.com/why-do-certificate-revocation-checking-mechanisms-never-work-f9b7a4ee1a61) (PKI). Cryptolyzer lagging behind competitors in this field.

<table><thead><tr><th>&nbsp;</th><th><strong>Crypto Lyzer</strong></th><th><a href="https://www.ssllabs.com/ssltest/">Qualys SSL Labs</a></th><th><a href="https://www.immuniweb.com/ssl/">Immuni Web</a></th><th><a href="https://www.hardenize.com/">Hard enize</a></th><th><a href="https://observatory.mozilla.org/">Moz. Obs.</a></th><th><a href="https://testssl.sh/">testssl .sh</a></th><th><a href="https://github.com/nabla-c0d3/sslyze">SSL-yze</a></th></tr></thead><tbody><tr><td><a href="https://datatracker.ietf.org/doc/html/rfc6066#section-8">OCSP Stapling</a></td><td>✅</td><td>✅</td><td>✅</td><td>✅</td><td>✅</td><td>✅</td><td>✅</td></tr><tr><td><a href="https://datatracker.ietf.org/doc/html/rfc7633">OCSP Must Staple</a></td><td><strong>0.8.x</strong></td><td>✅</td><td>✅</td><td>✅</td><td>❌</td><td>✅</td><td>✅</td></tr><tr><td><a href="https://certificate.transparency.dev/">Certificate Transparency</a></td><td><strong>0.8.x</strong></td><td>✅</td><td>✅</td><td>✅</td><td>❌</td><td>✅</td><td>✅</td></tr><tr><td><a href="https://en.wikipedia.org/wiki/Certificate_authority#Providers">Multiple Trusted Root CA Stores</a></td><td><strong>0.9.x</strong></td><td>✅</td><td>✅</td><td>✅</td><td>✅</td><td>❌</td><td>✅</td></tr><tr><td><a href="https://en.wikipedia.org/wiki/Public_key_infrastructure#Certificate_revocation">Revocation Status</a></td><td><strong>0.10.x</strong></td><td>✅</td><td>❌</td><td>✅</td><td>❌</td><td>✅</td><td>❌</td></tr><tr><td>&nbsp;</td><td><strong>20%</strong></td><td>100%</td><td>80%</td><td>100%</td><td>40%</td><td><strong>80%</strong></td><td>80%</td></tr></tbody></table>

### Generic[Permalink](#generic-1 "Permalink")

Usability is another field where Cryptolyzer should evolve. Comprehensiveness is not enough, weaknesses, vulnerabilities should be highlighted to make the analysis results more easily understandable.

<table><thead><tr><th>&nbsp;</th><th><strong>Crypto Lyzer</strong></th><th><a href="https://www.ssllabs.com/ssltest/">Qualys SSL Labs</a></th><th><a href="https://www.immuniweb.com/ssl/">Immuni Web</a></th><th><a href="https://www.hardenize.com/">Hard enize</a></th><th><a href="https://observatory.mozilla.org/">Moz. Obs.</a></th><th><a href="https://testssl.sh/">testssl .sh</a></th><th><a href="https://github.com/nabla-c0d3/sslyze">SSL-yze</a></th></tr></thead><tbody><tr><td>Client Emulation</td><td>❌</td><td>✅</td><td>❌</td><td>❌</td><td>✅</td><td>✅</td><td>❌</td></tr><tr><td>Weakness Check</td><td>❌</td><td>✅</td><td>✅</td><td>✅</td><td>✅</td><td>✅</td><td>❌</td></tr><tr><td>Vulnerability Check</td><td>❌</td><td>✅</td><td>✅</td><td>❌</td><td>❌</td><td>✅</td><td>✅</td></tr><tr><td>Highlights in Output</td><td>❌</td><td>✅</td><td>✅</td><td>✅</td><td>✅</td><td>✅</td><td>❌</td></tr><tr><td>&nbsp;</td><td>0%</td><td>100%</td><td>75%</td><td>50%</td><td>75%</td><td>100%</td><td>25%</td></tr></tbody></table>

Opportunities[Permalink](#opportunities "Permalink")
----------------------------------------------------

### DNS Records[Permalink](#dns-records "Permalink")

Nowadays information related to security settings is stored in DNS records especially, but not exclusively in the case of the SMTP protocol. These records can be analyzed by CryptoLyzer and the necessary implementations are planned to be done soon.

<table><thead><tr><th>&nbsp;</th><th><strong>Crypto Lyzer</strong></th><th><a href="https://www.ssllabs.com/ssltest/">Qualys SSL Labs</a></th><th><a href="https://www.immuniweb.com/ssl/">Immuni Web</a></th><th><a href="https://www.hardenize.com/">Hard enize</a></th><th><a href="https://observatory.mozilla.org/">Moz. Obs.</a></th><th><a href="https://testssl.sh/">testssl .sh</a></th><th><a href="https://github.com/nabla-c0d3/sslyze">SSL-yze</a></th></tr></thead><tbody><tr><td><a href="https://en.wikipedia.org/wiki/DNS_Certification_Authority_Authorization">DNS CAA</a></td><td>❌</td><td>✅</td><td>✅</td><td>✅</td><td>✅</td><td>✅</td><td>❌</td></tr><tr><td><a href="https://datatracker.ietf.org/doc/html/rfc8461">MTA-STS</a></td><td>❌</td><td>❌</td><td>❌</td><td>✅</td><td>❌</td><td>❌</td><td>❌</td></tr><tr><td><a href="https://en.wikipedia.org/wiki/DNS-based_Authentication_of_Named_Entities">DANE</a></td><td>❌</td><td>❌</td><td>❌</td><td>✅</td><td>❌</td><td>❌</td><td>❌</td></tr><tr><td><a href="https://en.wikipedia.org/wiki/Sender_Policy_Framework">SPF</a></td><td>❌</td><td>❌</td><td>❌</td><td>✅</td><td>❌</td><td>❌</td><td>❌</td></tr><tr><td><a href="https://en.wikipedia.org/wiki/DMARC">DMARC</a></td><td>❌</td><td>❌</td><td>❌</td><td>✅</td><td>❌</td><td>❌</td><td>❌</td></tr><tr><td><a href="https://datatracker.ietf.org/doc/html/rfc8460">TLS-RPT</a></td><td>❌</td><td>❌</td><td>❌</td><td>✅</td><td>❌</td><td>❌</td><td>❌</td></tr><tr><td>&nbsp;</td><td><strong>0%</strong></td><td>17%</td><td>17%</td><td><strong>100%</strong></td><td>17%</td><td>17%</td><td>0%</td></tr></tbody></table>

### Other protocols[Permalink](#other-protocols "Permalink")

Some cryptographic protocols use the variant of the TLS protocol. Some protocols wrap the original TLS protocol such as [OpenVPN](https://openvpn.net/community-resources/openvpn-cryptographic-layer/), [QUIC](https://datatracker.ietf.org/doc/rfc9001/), and there is a modified version of TLS, according to the need of UDP, called [DTLS](https://en.wikipedia.org/wiki/Datagram_Transport_Layer_Security). Their support would be implemented relatively easily in CryptoLyzer in the future.

Installation[Permalink](#installation "Permalink")
--------------------------------------------------

CryptoLyzer can be easily installed from [The Python Package Index](https://pypi.org/) (PyPi)

```
$ pip install cryptolyzer


```

or via Docker from [Docker Hub Container Image Library](https://hub.docker.com/) (docker hub)

```
$ docker pull coroner/cryptolyzer


```

and there are prebuilt packages for DEB (Debian, Ubuntu, …) and RPM (Fedora, RHEL, SUSE) based systems.

Usage[Permalink](#usage "Permalink")
------------------------------------

The command-line interface is slightly similar to the OpenSSL command-line tool. On each level of subcommands, comprehensive help can be given using the –help option. The analysis subject can be given multiple times as an argument in URL format. Most of the time scheme and port part of the URL has default values according to the analyzer (eg: tls, ssh).

```
$ cryptolyze tls all www.example.com

$ cryptolyze tls1_2 ciphers https://www.example.com:443

$ cryptolyze ssh all www.example.com

$ cryptolyze ssh2 ciphers ssh://www.example.com:22

$ cryptolyze http headers http://www.example.com/path?query#fragment


```

or can be used via Docker

```
$ docker run --rm coroner/cryptolyzer tls all www.example.com

$ docker run --rm coroner/cryptolyzer tls1_2 ciphers https://www.example.com:443

$ docker run --rm coroner/cryptolyzer ssh all www.example.com

$ docker run --rm coroner/cryptolyzer ssh2 ciphers ssh://www.example.com:22

$ docker run --rm coroner/cryptolyzer http headers http://www.example.com/path?query#fragment


```

[](http://creativecommons.org/licenses/by-sa/4.0/)CryptoLyzer: A comprehensive cryptographic settings analyzer is licensed under a [Creative Commons Attribution-ShareAlike 4.0 International License](http://creativecommons.org/licenses/by-sa/4.0/)