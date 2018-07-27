# A Draft Proposal for Local HTTPS

This document describes some technical approaches to achieve HTTPS communications between browsers and local servers that have private domain names such as ‘device.local’ or ‘device.home.arpa’ (hereafter, the local server is simply called ‘device’).

As the proposal is written from the standpoint of the users of browsers, it might not be based on the design policy, security policy, and current implementations of browsers, and also it is just an early stage draft proposal.
So, we kindly solicit comments on the basic concept and feasibility of the proposal from browser vendors, security and privacy experts, IoT device or web service developers.

# Introduction

The backgrounder (e.g., problem statements) behind the proposal is omitted because it has already been described in [the charter of the HTTPS in Local Network Community Group][1] and [the introduction of ‘HTTPS for Local Domains’][2].

The motivating use cases are written in [the CG’s github page][3]. Basically, we think there are two device access patterns for local HTTPS as follows:
- _Normal access pattern_: the device has web contents and a user types the address of the device (e.g., ‘https://device.local’) to the address bar of a browser directly and gets the contents.
- _Cross-origin access pattern_: the device has API endpoints and a web frontend loaded on a browser from the internet (hereafter, simply called ‘web service’) accesses the APIs with a browser API ([fetch API][4], etc.)

Most of the use cases listed on [the github page][3] are based on the latter access pattern because the CG focused on addressing a mixed content problem ([Mixed Content][5]) on the cross-origin access at first.
Therefore, all of the approaches proposed in later sections are assumed to be applied to the cross-origin access pattern.
However, some of the approaches can be applied to the normal access pattern and we discuss the topic [later](#consideration-for-normal-access-pattern).

# Scope of the Proposal and Existing Technologies

There are various means for browsers to allow HTTPS accesses to the devices. In terms of the server authentication, we focus on the following three approaches:
1.  Password（e.g., ephemeral PIN code displayed on the device）
2.  Self-singed certificate or raw public key
3.  Private CA  issued certificate (e.g., device vendor issued certificate)

On the other hand, we don't discuss the following approaches:
- Public CA  issued certificate
- [DANE][6]

The former is the well-known Web PKI approach. [Mozilla's IoT Gateway][7] and [PLEX][8] are known as existing solutions of this approach and each of them can be regarded as one of the best practices to realize the local HTTPS.
However, public CA cannot issue certificates for local domains such as .local or .home.arpa ([CA Browser Forum Guideline][9]).
This means that the devices have to use global unique domain names and it brings some drawbacks to the approach as follows:
- Short and memorable domain names like ‘device.local’ cannot be used.
- Devices must be accessible from the internet to get DV certificates from public CAs. It can be mitigated by setting routers to allow a specific server (e.g., Let’s Encrypt) to access the devices temporarilly. However, such kind of settings might be difficult for users in some use cases (e.g., for home networks).
- Accesses to devices will be failed when the internet is down because of the global name resolution, although the accesses are basically done in local networks.
- Domain names of devices are disclosed globally with their IP addresses. It might be ragarded as a kind of privacy issues.

In addtion to the above, the number of the certificates for the devices can be increasing significantly beyond the number of which the existing CAs have issued before.
That would also be one of the concerns of this approach. The approaches proposed in this document can be regarded as the means to solve the problem of this approach.
 
Ther latter is a [DANE][6]-based approach that uses DNS TLSA records to authenticate TLS servers.
In particular, DANE allows TLS servers to use ‘domain-issued certificates’ that can be issued by a domain administrator wihout involving a third-party CA.
That sounds useful for local HTTPS. However, DANE is not supported on browsers so far and the approach also needs DNSSEC support for local network.
So, we think the approach is not feasible relatively (compared with the proposed approaches below) for now and we desiced to put it on out-of-scope at the time being.

# Overview of the Proposal

In this section, we describe the outlines of the approach #1, #2, and #3.

_NOTE: For the following discussion, we use ‘device.local’ as an example of a domain name of a device. Of course, other local domain names (e.g., ‘device.home.arpa’) should be applicable and the name resolution methods shouldn’t be restricted to [mDNS][10]._

## Approach #1 (J-PAKE-based Approach)

When a web service (e.g., https://device-user.example.com) accesses a device (e.g., device.local) that has a self-singed certificate via a browser, we think that the most simple approach is that the browser allows the access only if the user grants the access through the UI shown the figure below, which is displayed when fetch API is called for the access.

<div align="center">
<img src="https://github.com/dajiaji/proposals/blob/figures/figs/fig_approach_1_1.png" width="480px">
</div>

However, it cannot guarantee whether the ‘device.local’ displayed on the pop-up window is really the same as the domain name of the device which the user intends to grant the access to.
If an evil device on the local network gets a plausible domain name like ‘printer.local’ upfront and then the proper device has to use a secondary name like ‘printer2.local’, it may be difficult for the user to find such a situation.

Approach #1 resolves this problem by extending a browser API and UI. Specifically, by adding a textbox to input a PIN code to the pop-up window as follows, it makes the users to be able to bind the displayed domain name to the real device.

<div align="center">
<img src="https://github.com/dajiaji/proposals/blob/figures/figs/fig_approach_1_2.png" width="480px">
</div>

Since the PIN code should be short and easy to input from the aspect of usability, we believe that it is a good choice to use [J-PAKE][11], which provides the way to generate and share a cryptographic secure key based on a short password.
If browsers can support [Elliptic Curve J-PAKE Cipher Suites for TLS][12], the above UI can be implemented in more secure manner.
In addition, J-PAKE has already been implemented on Firefox and the use of J-PAKE has been discussed in [W3C Second Screen WG][13] too. From this point of view, we believe that the approach is basically practical.

_NOTE: As [ECJPAKE][12] defines only the way to generate a shared key based on a password,  we have to define a specific way to bind the domain name to the TLS session. For example, using ‘local domain name + fingerprint of the self-signed certificate’ as the TLS server identifier might be one of the solution._
Supporting J-PAKE, fetch API might have to have an additional parameter that to declare the use fo J-PAKE explicitly as follows.

```javascript
fetch("https://device.local/stuff",
  {tls: "jpake"})
.then(res => res.json());
```

Besides using ephemeral PIN code displayed on a device, there are other ways to share a password as follows:
- Use a secondary communication chanel such as BLE or NFC.
- Use a static PIN code printed on the device.

### Requirements for Browsers 

Regarding Approach #1, the requirements for browsers can be summarized as follows:
- Support an additional parameter of fetch API (e.g., tls: ‘jpake’)
- Support additional cipher suites for J-PAKE (TLS\_ECJPAKE\_\* defined in [ECJPAKE][12])
- Implement the pop-up window shown above.

In addition to the above, we might have to consider the way to use a self-signed certificate after the first J-PAKE-based TLS session to mitigate the every-time PIN code input.
If a TLS server uses ‘the domain name + a fingerprint of the self-signed certificate’ as the server identifier,the browser might be able to trust the self-signed certificate on subsequent TLS sessions, based on the identifier.

### Technical Topics for Other SDOs

Regarding Approach #1, following technical topics might have to be discussed in other SDOs:
- [ECJPAKE](ECJPAKE): This document is merely an individual submission and it has been expired since two years ago. We don’t know the current status of the document but it might be necessary to resume the standardization activity.
- A method to bind a domain name to a TLS session over ECJPAKE (As described above, using ‘local domain name + fingerprint of the self-signed certificate’ as the TLS server identifier might be one of the solution).

## Approach #2 (ACE/OAuth based approach)

Approach #1 basically depends only on user approval. There are no trust anchors that can guarantee the authenticity of devices and it is difficult for users to find whether the device is infected by a malware or not from its appearance.
From the standpoint of web services, it looks unreliable that a server authentication must be delegated only to the user’s judgement.

Approach #2 resolves this problem by introducing an [ACE][14]/[OAuth][15] AS (Authorization Server) as an authority of the device into the local HTTPS system based on the ACE framework shown the figure below.

<div align="center">
<img src="https://github.com/dajiaji/proposals/blob/figures/figs/fig_approach_2_1.png" width="480px">
</div>

The ACE components have following relationships:
- The device, which is regarded as an RS (Resource Server) in the context of ACE, has a trust relationship with the AS. In our assumption, the AS is basically maintained by the device manufacturer.
- The web server, which is regarded as a client in the context of ACE, has been registered to the AS in advance and has a trust relationship with it.
- The user, which is regarded as a Resource Owner in the context of ACE, has an account for the AS and the ownership information to device has been registered to the AS in advance.
Under the relationship, the client can get an access token to access the device based on the user approval. At this time, the client can also get the RS information that includes an URI and an RPK (Raw Public Key) or self-signed certificate of the device (step (B) shown the above). However, existing web browsers don’t permit the access to the device with the access token (step (C)). 

Therefore, Approach #2 extends a browser API and related UI to enable the client to access the device only when the client provides the browser with the RS information (a self-signed certificate or RPK) as a trusted one. The RS information can be sent to the browser as an extended parameter of fetch API as follows:

```javascript
// When RS Information includes a RPK, 
fetch("https://device.local/stuff", {
  tls: "rpk",
  certificate: "base64-encoded rpk>"})
.then(res => res.json());

// When RS Informatin includes a self-signed certificate, 
fetch("https://device.local/stuff", {
  tls: "pkix", // default value that can be omitted.
  certificate: "<base64-encoded certificate or its fingerprint>"})
.then(res => res.json());
```

 When the fetch API above is called, the browser shows a following pop-up window.

<div align="center">
<img src="https://github.com/dajiaji/proposals/blob/figures/figs/fig_approach_2_2.png" width="480px">
</div>

As the result, HTTPS accesses from the web service to the device are allowed by the browser based on the trust relationship between the client and the AS (and the device) and, user aapproval.

In addtion, Approach #2 has another advantage that the device (RS) has an opportunity to know the client(‘s origin) before the access from the device because, not to mention, the access token is issued before the access.
This means that the device can use proper CORS settings for the client dynamically. The feasure has an affinity for a secure local cross-origin access method described in [CORS and RFC1918][16].

### Requirements for Browsers 

Regarding Approach #2, the requirements for browsers can be summarized as follows:
- Support additional parameters of Fetch API mentioned above.
- Support a TLS certificate type and extensions for using RPK defined in [Using RPK in TLS and DTLS][17]
- Implement the pop-up window shown above.

### Technical Topics for Other SDOs

Regarding Approach #2, following technical topics might have to be discussed in other SDOs:
- HTTPS profile for ACE (or extensions for OAuth): As ACE is basically based on CoAPS/OSCORE, it is necessary for the approach to define HTTPS profile for ACE or to define some extensions under the assumption that the RSs are distributed and deployed in local network.

## Approach #3

Approach #2 can be regarded as an application-layer solution. Therefore, when we want to have a revocation mechanism ready in case that the devices’ severe vulnerability is disclosed, we have to implement it on top of Approach #2. If we can adopt a TLS-layer solution, we can use existing technologies of certificate revocation (CRL, OCSP Stapling, etc.).

We believe that the framework, on which a device vendor validates domain names of the devices and guarantees the authenticity of them, would be useful even if the names are ununique local names.
Actually, there are some standardization activities related to such a vendor-issued certificte (hereafter, private CA issued certificate).
For example, [IEEE802.1AR][18] defines IDevID and LdevID that are device identifiers issued by the device manufacturers, and it seems that [PKI Certificate Identifier Format for Devices][19] tries to make the device identifiers be available on the Web PKI.
In addition, [IETF ANIMA WG][20] has been developing a way to issue LDevID, which is a sort of vendor-issued certificates, based on IDevID.
Therefore, the vendor-issued certificate can be regarded as an ordinary concept on IoT security and its trust model.
Furthermore, vendor-issued certificates are useful to guarantee the devices have TPM functionarity, which is a candidate solution to keep private keys secure even if the devices are in local network that has no administrators.

However, there is no way to use the trust chain based on the vendor-issued certificates on existing Web PKI seamlessly for now.
Approach #3 extends a browser API and a related UI to support vendor-issued certificates (generally, private CA issued certificate) in a similar way to Approach #2.
Specifically, by extending the fetch API as follows, it enables web services to provide the browsers with private CA certificates or private CA issued certificates as trusted ones.

```javascript
fetch("https://device.local/stuff", {
  tls: "pkix", // default value that can be omitted.
  certificate: "<base64-encoded certificate or its fingerprint>"})
.then(res => res.json());
```

As with approach #1 and #2, the browser shows a following pop-up window when the fetch API above is called.

<div align="center">
<img src="https://github.com/dajiaji/proposals/blob/figures/figs/fig_approach_3_1.png" width="480px">
</div>

Approach #3 can be built on the top of Approach #2. This means that the AS has a private CA role and the web service trusts a private CA issued certificate based on the pre-registered relationship with the CA/AS. Of course, it is not necessary for the web service to make the trust relationship in advance. The web service can be assumed that it trusts the private CA without any rationale.

In addition, we proposed a method to issue a DV(Domain Validation) certificate for a device at [a breakout session in TPAC 2017][21].
The method is based on the OOB (Out-of-Band) challenge defined in the previous draft of [ACME][22].
The challenge, which is the access to ‘https://device.local/.well-known/acme-challenge/{token}’, is executed by the ACME server’s frontend loaded on a browser that can communicate with the device in local network.
Although the challenge through the browser has some advantages (e.g., it can be based on user approval, there is no need to change firewall settings), it requires a fetch API extension for approach #2 that enables the access to the device based on self-signed certificates or RPKs.

We are very pleased to hear the comments and reviews on the feasibility of the ACME extension for ‘Let’s Encrypt for devices’.

### Requirements for Browsers 

Regarding Approach #3, the requirements for browsers can be summarized as follows:
- Support additional parameters of fetch API mentioned above.
- Implement the pop-up window shown above.

### Technical Topics for Other SDOs

Regarding Approach #3, following technical topics might have to be discussed in other SDOs:
- [PKI Certificate Identifier Format for Devices][19]: This has been an individual submission yet.
- The ACME extension described above.

# Consideration for Normal Access Pattern

As for the normal access pattern mentioned above, it is not necessary to extend fetch API. In addtion, the proposals for approach #2 are not useful for the normal access pattern because approach #2 is inherently based on the cross origin access. 

On the other hand, we believe that the UI extensions and additional protocol supports for approach #1 and #3 can also be candidate proposals for the normal access pattern.
Of course, there are a few things to consider to apply to the normal access pattern. For example, we have to clarify the means for browsers to get private CA certificates for validating device certificates.

# Conclution

We proposed three approaches to achieve local HTTPS without using public CA certificates.
All of them are nothing more than early stage ideas. We’d like to refine the proposal and move forward with the CG activity through open discussions.

<!-- References -->

[1]: https://httpslocal.github.io/cg-charter/
[2]: https://docs.google.com/document/d/170rFC91jqvpFrKIqG4K8Vox8AL4LeQXzfikBQXYPmzU/edit?usp=sharing
[3]: https://github.com/httpslocal/usecases
[4]: https://fetch.spec.whatwg.org/
[5]: https://www.w3.org/TR/mixed-content/
[6]: https://tools.ietf.org/html/rfc6698
[7]: https://iot.mozilla.org/gateway/
[8]: https://blog.filippo.io/how-plex-is-doing-https-for-all-its-users/
[9]: https://cabforum.org/wp-content/uploads/Guidance-Deprecated-Internal-Names.pdf
[10]: https://tools.ietf.org/html/rfc6762
[11]: https://tools.ietf.org/html/rfc8236
[12]: https://tools.ietf.org/html/draft-cragie-tls-ecjpake-01
[13]: https://www.w3.org/2014/secondscreen/
[14]: https://datatracker.ietf.org/wg/ace/about/
[15]: https://datatracker.ietf.org/wg/oauth/about/
[16]: https://wicg.github.io/cors-rfc1918/
[17]: https://tools.ietf.org/html/rfc7250
[18]: https://1.ieee802.org/security/802-1ar/
[19]: https://tools.ietf.org/id/draft-friel-pki-for-devices-00.html
[20]: https://datatracker.ietf.org/wg/anima/about/
[21]: https://www.w3.org/wiki/File:TPAC2017_httpslocal-2.pdf
[22]: https://tools.ietf.org/html/draft-ietf-acme-acme-12
[HomeArpa]: https://tools.ietf.org/html/rfc8375
[Voucher]: https://tools.ietf.org/html/rfc8366

