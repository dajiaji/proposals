# Supporting Local HTTPS Communication - A Draft Proposal 

# 1. Scope

This document describes candidate technical approaches to achieve HTTPS communication between a browser (HTTP client) and a local server (HTTP server) that has private domain names such as ‘server.local’ or ‘server.home.arpa’. For the purpose of this document, we assume that the browser is hosted in a user entity (e.g., Smart Phone) and the local server is hosted in a home device. Therefore, we use the term ‘device’ instead of a local server.

# 2. Purpose

The purpose of this draft proposal is to initiate discussions and receive feedbacks from the W3C members especially from the browser vendors and web developers. Authors of this document are aware that the proposed approaches might not be based on the current design and security policies of the browser implementations but certainly hope that the community will update these policies to cater the need for new emerging markets, such as Internet of Things (IoT).

# 3. Use Cases

The motivation and use cases are described in [the charter of the HTTPS in Local Network Community Group (CG)][1]. These use cases are developed under [the charter of the HTTPS in Local Network CG][2]. In general, we can categorize two device access patterns principle for local HTTPS. They are as follows:
- _Normal access pattern_: the device has web contents and a user types the address of the device (e.g., ‘https://device.local’) on the browser directly and receives the contents.
- _Cross-origin access pattern_: the device has API endpoints and a web frontend loaded on a browser from the internet (hereafter, simply called ‘web service’) accesses the APIs with a browser API (e.g., [fetch API][3]) and receives the contents.

Most of the use cases listed in [the github page][1] however are based on cross-origin access pattern because the Community Group mainly focused on addressing a mixed content problem ([Mixed Content][4]). Therefore, all approaches proposed in this draft are based on the cross-origin access pattern principle although some of the approaches can be applied to the normal access pattern as well. 

# 4. Existing Techniques

## 4.1. Web PKI-based Approach

In this approach, public CA issues certificates for devices that have a globally unique domain names because public CA cannot issue certificates for local domains such as .local or .home.arpa as described in [CA Browser Forum Guideline][20]. Example current industry best practices of this approach are [Mozilla's IoT Gateway][5] and [PLEX][6]. While these approaches are deployed and fits well in some use cases, they cannot work in scenarios where domains are for example, ‘server.local’ or ‘server.home.arpa’. Moreover, the access to local devices in the first place is an important issue (we understand that this issue can be mitigated by preconfiguring the gateway router/proxy but these settings are difficult to be made available in scenarios such as home networks). The other issue is resolving the domain name when there is no Internet access. Finally, the number of required public CAs is significantly larger than other use cases due to the vast nature of IoT devices.

## 4.2. DANE-based Approach

This approach ([DANE][7]) allows TLS servers to use ‘domain-issued certificates’ that can be issued by a domain administrator without involving a third-party CA. While this technique seems useful for local HTTPS, DANE is not supported in major browsers. In addition, the approach needs DNSSEC support for local network. Therefore, we argue that DANE-based approach is difficult to realize in support of the IoT use cases.

# 5. Overview of the Proposed Technical Approaches

In this section, we describe three technical approaches for communications between a browser (HTTP client) and a local server (HTTP server) that has private domain names.

_NOTE: For the following discussions, we use ‘device.local’ as an example of a domain name of a device. Of course, other local domain names (e.g., ‘device.home.arpa’) are applicable name which can be resolved but the name resolution methods should not be restricted only to mDNS_.

## 5.1. Approach #1

This approach is based on the user grant and the use of shared password in which [J-PAKE: Password-Authenticated Key Exchange by Juggling][8] is used for the establishment of a secure end-to-end communication channel between the user device and the local server. It is worthwhile to mention that J-PAKE has already been implemented in [Mbed TLS][9] and also the use of J-PAKE has been discussed in [W3C Second Screen WG][10]. Using the following methods, this approach can be realized:

A web service (e.g., https://device-user.example.com) accesses a device (e.g., ‘device.local’) via a browser. The browser allows the access only if the user grants the access through the UI shown in the figure below. The UI will be displayed when the underlying fetch API is called for the access and the device successfully performs a TLS handshake using [Elliptic Curve J-PAKE Cipher Suites][11] (or, other PAKE-based cipher suites).

<div align="center">
<img src="https://github.com/dajiaji/proposals/blob/figures/figs/fig_approach_1_2.png" width="480px">
</div>

To make sure that the ‘device.local’ displayed on the pop-up window is really the same as the domain name of the device which the user intends to grant the access to, user inserts either a PIN or password  through the pop-up window.

The above flow can be achieved by extending the UI with adding the cipher suites in the browser. This will enable binding the displayed domain name to the physical device.

_NOTE: However, a specific method on how to bind the name to the TLS session needs to be defined to distinguish devices which have the same names as mentioned in [HTTPS for Local Domains][22]. This method is needed for both approaches #2 and #3. For example, using device ‘local domain name + fingerprint in self-signed certificate’ as the TLS server identifier (which is defined in [Elliptic Curve J-PAKE Cipher Suites][11]) can be considered as one of the candidate solutions. The use of self-signed certificate after the first J-PAKE based TLS session needs to be defined as well. This will mitigate the PIN or password entry for subsequent TLS sessions._

### 5.1.1. Browser Requirements

The requirements for browsers can be summarized as follows:
- Support additional cipher suites for J-PAKE.
- Implement the pop-up window for PIN/Password input.
- Support a method to use a self-signed certificate after the first J-PACKE-based TLS session.

### 5.1.2. Dependency to other SDOs

This approach will require work and collaboration with the IETF.
- [Draft document][11] was an individual submission and it is currently expired. If W3C embraces this solution, the work needs to resumed and completed.
- A method to bind a domain name to a TLS session over ECJPAKE needs to be specified and standardized.

## 5.2. Approach #2

In approach #1, there is no trust anchor that can guarantee the authenticity of devices and it is difficult for users to find whether the device is a legitimate one. From the standpoint of web services, it is often argued whether a server authentication should be delegated to a user’s judgement.

Approach #2 resolves this problem by introducing an [ACE][12] and [OAuth][13] based AS (Authorization Server) as an authority of the device into the local HTTPS system as shown below.

<div align="center">
<img src="https://github.com/dajiaji/proposals/blob/figures/figs/fig_approach_2_1.png" width="480px">
</div>

The ACE components have the following relationships:
- The device, which is regarded as an RS (Resource Server) in the context of ACE, has a trust relationship with the AS. In our assumption, the AS is configured by the device manufacturer.
- The web server, which is regarded as a client in the context of ACE, has been registered to the AS in advance and has a trust relationship with it.
- The user, which is regarded as a Resource Owner in the context of ACE, has an account for the AS and the ownership information to device has been registered to the AS in advance.

Under the above relationships, the client (Web Service) can get an access token to access the device based on user’s approval. At this time, the client can also get the RS information that includes an URI and an [RPK (Raw Public Key)][14] or self-signed certificate of the device (step (B) as shown in above figure). Since the existing web browsers do not permit the access to the device with the access token (step (C) in above figure), browser API and related UI need to be extended to enable the client to access the device only when the client provides the browser with the RS information (a self-signed certificate or RPK) as a trusted one.

The RS information can be sent to the browser as an extended parameter of fetch API as follows:

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

As the result, HTTPS accesses from the web service to the device are achieved by the browser based on the trust relationship between the client and the AS (and the device) and with user’s explicit approval.

In addition, when the trust relationship between AS and the devices is built on attestation keys in TPM (Trusted Platform Module) on the devices, the authenticity of the devices can be enhanced. The use of the attestation keys is applicable to Approach #3 as well.

### 5.2.1. Browser Requirements

The requirements for browsers can be summarized as follows:
- Support additional parameters of fetch API mentioned above.
- Support a TLS certificate type and extensions for using [RPK][14].
- Implement the pop-up window shown above.

### 5.2.2. Dependency to other SDOs

This approach will require work and collaboration with the IETF.
- HTTPS profile for ACE (or extensions for OAuth): Since ACE is based on CoAP/OSCORE, it is necessary to define either a HTTPS profile for ACE or an extension to COAP/OSCORE under the assumption that the RSs are distributed and deployed in local network.

## 5.3. Approach #3

In this approach, the web service trusts a vendor issued private certificate and asks the web browser to pin the certificate as a trusted one (only in the web service’s secure context) based on user grant. The trust can be built as an extension to Approach #2 in which AS has a private CA role and the web service trusts a private CA issued certificate based either on a pre-registered relationship with the CA and AS or by some other means.

This approach can be realized by extending the browser API and related UI in a similar way as shown in Approach #2. Specifically, by extending the fetch API as described below, it enables web services to provide the browsers with private CA certificates or private CA issued certificates as trusted ones.

```javascript
fetch("https://device.local/stuff", {
  tls: "pkix", // default value that can be omitted.
  certificate: "<base64-encoded certificate or its fingerprint>"})
.then(res => res.json());
```

Similar to earlier approaches, the browser shows a following pop-up window when the fetch API above is called.

<div align="center">
<img src="https://github.com/dajiaji/proposals/blob/figures/figs/fig_approach_3_1.png" width="480px">
</div>

For this approach, we argue that the framework on which a device vendor validates domain names of the devices and guarantees the authenticity of them would be useful even if the names are local names. A domain-validated certificate can be issued by using the OOB (Out-of-Band) challenge as defined in [the earlier draft of ACME][15]. This was also discussed in [TPAC 2017 breakout session][16]. The challenge, which is the access to ‘https://device.local/.well-known/acme-challenge/{token}’, is executed by the ACME server’s frontend loaded on a browser that can communicate with the device in a local network. Although the challenge through the browser has some advantages (e.g., it can be based on a user grant, there is no need to change the firewall settings), it requires a fetch API extension for approach #2 that enables the access to the device based on self-signed certificates or RPKs.

It is important to note that there are other industry efforts on similar concepts as vendor-issued certificate (hereafter, private CA issued certificate). For example, [IEEE802.1AR][17] defines IDevID and LDevID that are device identifiers issued by the device manufacturers, and it seems that [PKI Certificate Identifier Format for Devices][18] tries to make the device identifiers be available on the Web PKI. In addition, [IETF ANIMA WG][19] has discussed the way to issue an LDevID autonomously based on an IDevID. Therefore, we propose that the vendor-issued certificate should be regarded as an accepted mechanism that should be leveraged by the W3C community. 

### 5.3.1.  Browser Requirements

The requirements for browsers can be summarized as follows:
- Support additional parameters of fetch API mentioned above.
- Implement the pop-up window shown above.

### 5.3.2.  Dependency to other SDOs

This approach will require work and collaboration with the IETF.
- [PKI Certificate Identifier Format for Devices][18] is currently an individual submission yet. 
- The ACME extension as described above.

## 5.4.  Pros and Cons of the Approaches

###  Approach #1

- Pros 
    - There is no need for manufactures to deploy and maintain their own servers (AS and/or CA) on the internet.
    - It is applicable to both access patterns use cases.
    - There is no need to extend the Fetch API.
- Cons
    - There is no trust anchor for web services to trust the devices and their domain names.
    - A user has to input a PIN /password, or a device has to support a secondary communication channel (e.g., BLE, NFC).
    - Web browsers need to support PAKE-based cipher suites.

### Approach #2

- Pros
    - Web services can trust devices as far as they can trust AS for the devices.
    - If a device can get web service information from the AS, the device can configure proper CORS settings in advance. It means that the approach would be familiar with the secure local cross-origin access method described in [CORS and RFC1918][21]).
    - The authenticity of devices can be enhanced when the AS authenticate devices based on attestation keys in TPM on the devices.
- Cons
    - Manufactures have to deploy and maintain their own servers (AS and/or CA).
    - Fetch API needs to be extended.
    - Device domain names are not validated.
    - Another browser API (for example, which allows a web service to pin a certificate in the context of a specific origin) would be needed to support the normal access pattern use case.

### Approach #3

- Pros
   - Web services can trust devices as far as they can trust private CAs for the devices.
   - Device domain names can be validated if ACME can be extended for local domain names.
   - Existing PKI-based methods for managing the lifecycle of certificates can be used (e.g., CRL, OCSP).
   - If a device can get web service information from the AS which has a private CA role, the device can configure proper CORS settings in advance as with Approach #2.
   - The authenticity of devices can be enhanced when the CA issues certificates for the devices based on attestation keys in TPM on the devices.
- Cons   
   - Manufactures have to deploy and maintain their own servers (AS and/or private CA).
   - Fetch API needs to be extended.
   - Another browser API would be needed to support the normal access pattern use case.

# 6. Conclusion

In this document, we introduced three approaches to enable local HTTPS communications that would not require the browsers having public CA certificates. However, these approaches would require browser extensions and other protocol standardization as identified. The motivation came from the industry need of providing HTTPS access to IoT devices, majority of which will not have a global domain name so that the domain name can be verified and a certificate can be issued. We hope that this paper will spur the discussions within the Community Group and in general within the W3C community so that a solution can be developed and standardized.

<!-- # A.  References -->

[1]: https://github.com/httpslocal/usecases
[2]: https://httpslocal.github.io/cg-charter/
[3]: https://fetch.spec.whatwg.org/
[4]: https://www.w3.org/TR/mixed-content/
[5]: https://iot.mozilla.org/gateway/
[6]: https://blog.filippo.io/how-plex-is-doing-https-for-all-its-users/
[7]: https://www.internetsociety.org/resources/deploy360/dane/ 
[8]: https://tools.ietf.org/html/rfc8236
[9]: https://www.mbed.com/en/technologies/security/mbed-tls/ 
[10]: https://www.w3.org/2014/secondscreen/
[11]: https://tools.ietf.org/html/draft-cragie-tls-ecjpake-01
[12]: https://datatracker.ietf.org/wg/ace/about/
[13]: https://datatracker.ietf.org/wg/oauth/about/
[14]: https://tools.ietf.org/html/rfc7250
[15]: https://tools.ietf.org/html/draft-ietf-acme-acme-8
[16]: https://www.w3.org/wiki/File:TPAC2017_httpslocal-2.pdf
[17]: https://1.ieee802.org/security/802-1ar/
[18]: https://tools.ietf.org/id/draft-friel-pki-for-devices-00.html
[19]: https://datatracker.ietf.org/wg/anima/about/
[20]: https://cabforum.org/wp-content/uploads/Guidance-Deprecated-Internal-Names.pdf
[21]: https://wicg.github.io/cors-rfc1918/
[22]: https://docs.google.com/document/d/170rFC91jqvpFrKIqG4K8Vox8AL4LeQXzfikBQXYPmzU/edit?usp=sharing