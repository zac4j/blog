---
title: "What is HTTPS"
date: 2022-03-12T20:36:33+08:00
description: "HTTPS in a nutshell"
tags: ["https", "tls", "ssl"]
categories: ["network"]
author: "Zac"
draft: false
---

HTTPS(Hypertext transfer protocol sercure) is a secure version of HTTP protocol that uses the [SSL/TLS protocol][sp] for encryption and authentication. HTTPS is specified by [RFC 2818][rfc-2818] and uses port 443 as default port(which HTTP is 80).

<!-- more  -->

## Why HTTPS is important

The HTTPS protocol makes it possible for website users to transmit sensitive data such as credit card numbers, banking information, and login credentials securely over the internet. For this reason, HTTPS is especially important for securing online activities such as shopping, trading and banking.

## Why HTTPS safer than HTTP

HTTPS adds **encryption**, **authentication**, and **integrity** to the HTTP protocol:

+ **Encryption**: HTTP is a clear text protocol, while HTTPS including SSL/TLS encryption, HTTPS prevents data sent over the internet from being intercepted and read by a third party. Through [public-key cryptography][pkc](*asymmetric cryptography*) and the [SSL/TLS handshake][hs], an encrypted communication session can be securely set up between two parties via the creation of a shared secret key.
+ **Authentication**: A website's SSL/TLS certificate includes a **public key** that a client(Browser) can use to confirm that data sent by the server have been digitally signed by **private key**. If the server's certificate is signed by a public trusted [certificate authority][ca](CA), the browser will accept that any identifying information included in the certificate has been validated by a trusted third party.
  + HTTPS websites can also be configured for [mutual authentication][ma], in which a web browser persents a client certificate identifying the user. Mutual authentication is useful for situations such as remote work, where it is desirable to include multi-factor authentication, reducing the risk of phishing or other attacks involving credential theft.
+ **Integrity**: Each document(such as HTML, CSS, or JavaScript file) sent to browser by a HTTPS web server includes a digital signature that a web browser can use to determine that the document has not been altered by a third party or otherwise corrupted while in transit. The server calculates a cryptographic hash of the document's contents, included with its digital certificate, which the browser can independently calculate to provce that the document's integrity.

---

### SSL/TLS handshake

![tls_handshake](/img/tls_handshake.png)

## How does HTTPS work

HTTPS adds encryption to the HTTP protocol by wrapping HTTP inside the SSL/TLS protocol(this is why SSL is called a tunneling protocol), so that all messages are encrypted in both directions between two connected computers. Although an attacker can still potentially access IP addresses, port numbers, domain names, the amount of information exchanged, and the duration of a session, all of the actual data exchanged are securely encrypted by SSL/TLS, including:

+ Request URL
+ Website content
+ Query parameters
+ Cookies

HTTPS also uses the SSL/TLS protocol for authentication SSL/TLS uses digital documents known as [X.509 certificates][x509](a public key certificates standard format) to bind cryptographic key pairs to the identities of entities such as websites, individuals, and companies. Each key pair includes a **private key**, and a **public key**, any one with public key can use it to:

+ Send a message that only the possessor of the private key can decrypt.
+ Confirm that a message has been digitally signed by its corresponding private key.

If the certificate presented by an HTTPS website has been signed by a publicly trusted **certificate authority(CA)**, such as SSL.com, users can be assured that the identity of the website has been validated by a trusted third party.

https://www.ssl.com/faqs/what-is-https/
https://httptoolkit.tech/blog/intercepting-android-https/
https://www.reddit.com/r/androiddev/comments/jqwbu9/ssl_decryption/

[sp]:https://www.ssl.com/faqs/faq-what-is-ssl/
[rfc-2818]:https://tools.ietf.org/html/rfc2818
[hs]:https://www.ssl.com/article/ssl-tls-handshake-overview/
[ca]:https://www.ssl.com/faqs/what-is-a-certificate-authority/
[ma]:https://www.ssl.com/how-to/configuring-client-authentication-certificates-in-web-browsers/
[x509]:https://www.ssl.com/faqs/what-is-an-x-509-certificate/
