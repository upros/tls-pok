---

title: "Bootstrapped TLS Authentication"
abbrev: TLS-POK
docname: draft-ietf-emu-bootstrapped-tls-01
category: std
ipr: trust200902

stand_alone: yes
pi: [toc, sortrefs, symrefs]

author:
 -
    ins: O. Friel
    name: Owen Friel
    org: Cisco
    email: ofriel@cisco.com
 -
    ins: D. Harkins
    name: Dan Harkins
    org: Hewlett-Packard Enterprise
    email: daniel.harkins@hpe.com
informative:

  IEEE802.1X:
    author:
      ins: IEEE
      name: IEEE
      org: IEEE 
    title: Port-Based Network Access Control
    date: 2010

  DPP:
    author:
      org: Wi-Fi Alliance
    title: Device Provisioning Profile
    date: 2020

  duckling:
    author:
      -
        ins: F. Stajano
        name: Frank Stajano
      -
        ins: E. Rescorla
        name: Eric Rescorla

    title: "The Ressurecting Duckling: Security Issues for Ad-Hoc Wireless Networks"
    date: 1999

--- abstract

This document defines a mechanism that enables a bootstrapping device to establish trust and mutually authenticate against a network. Bootstrapping devices have a public private key pair, and this mechanism enables a network server to prove to the device that it knows the public key, and the device to prove to the server that it knows the private key. The mechanism leverages existing DPP and TLS standards and can be used in an EAP exchange.

--- middle


# Introduction

On-boarding of devices with no, or limited, user interface can be difficult.  Typically, a credential is needed to access the network
and network connectivity is needed to obtain a credential.  This poses a catch-22.

If a device has a public / private keypair, and trust in the integrity of a device's public key can be obtained in an out-of-band fashion, a device can be authenticated and provisioned with a usable credential for network access.  While this authentication can be strong, the device's authentication of the network is somewhat weaker.  [duckling] presents a functional security model to address this asymmetry.

Device on-boarding protocols such as the Device Provisioning Profile [DPP], also referred to as Wi-Fi Easy Connect, address this use case but they have drawbacks. [DPP] for instance does not support wired network access.  This document describes an on-boarding protocol that can be used for wired network access, which we refer to as TLS Proof of Knowledge or TLS-POK.

## Terminology

The following terminology is used throughout this document.

- 802.1X: IEEE Port-Based Network Access Control

- BSK: Bootstrap Key which is an elliptic curve public private key pair

- DPP: Device Provisioning Protocol [DPP]

- EAP:  Extensible Authentication Protocol {{?RFC3748}}

- EPSK: External Pre-Shared Key

- EST: Enrollment over Secure Transport {{?RFC7030}}

- PSK: Pre-Shared Key

- TEAP: Tunnel Extensible Authentication Protocol {{RFC7170}}

## Bootstrapping Overview

A bootstrapping device holds a public / private key pair which we refer to as a Bootstrap Key (BSK). The private key of the BSK is known only by the device. The public key of the BSK is known by the device, is known by the owner or holder of the device, and is provisioned on the network by the network operator. In order to establish trust and mutually authenticate, the network proves to the device that it knows the public part of the BSK, and the device proves to the network that it knows the private part of the BSK. Once this trust has been established during bootstrapping, the network can provision the device with a credential that it uses for subsequent network access. More details on the BSK are given in {{bootstrap-key-pair}}.

## EAP Network Access

Enterprise deployments typically require an [IEEE802.1X]/EAP-based authentication to obtain network access. Protocols like Enrollment over Secure Transport (EST) {{?RFC7030}} can be used to enroll devices into a Certification Authority to allow them to authenticate using 802.1X/EAP. This creates a Catch-22 where a certificate is needed for network access and network access is needed to obtain certificate.

Devices whose BSK public key can been obtained in an out-of-band fashion and provisioned on the network can perform an EAP-TLS-based exchange, for instance Tunnel Extensible Authentication Protocol (TEAP) {{?RFC7170}}, and authenticate the TLS exchange using the bootstrapping extensions defined in {{bootstrapping-in-tls-13}}. This network connectivity can then be used to perform an enrollment protocol (such as provided by {{?RFC7170}}) to obtain a credential for subsequent network connectivity and certificate lifecycle maintenance.

# Bootstrap Key Pair

The mechanism for on-boarding of devices defined in this document relies on bootstrap key pairs. A client device has an associated elliptic curve (EC) bootstrap key pair (BSK). The BSK may be static and baked into device firmware at manufacturing time, or may be dynamic and generated at on-boarding time by the device. If the BSK public key, specifically the ASN.1 SEQUENCE SubjectPublicKeyInfo from {{!RFC5280}}, can be shared in a trustworthy manner with a TLS server, a form of "origin entity authentication" (the step from which all subsequent authentication proceeds) can be obtained. 

The exact mechanism by which the server gains knowledge of the BSK public key is out of scope of this specification, but possible mechanisms include scanning a QR code to obtain a base64 encoding of the ASN.1-formatted public key or uploading of a Bill of Materials (BOM) which includes the public key. If the QR code is physically attached to the client device, or the BOM is associated with the device, the assumption is that the public key obtained in this bootstrapping method belongs to the client. In this model, physical possession of the device implies legitimate ownership.

The server may have knowledge of multiple BSK public keys corresponding to multiple devices, and existing TLS mechanisms are leveraged that enable the server to identity a specific bootstrap public key corresponding to a specific device.

Using the process defined herein, the client proves to the server that it has possession of the private key of its BSK. Provided that the mechanism in which the server obtained the BSK public key is trustworthy, a commensurate amount of authenticity of the resulting connection can be obtained. The server also proves that it knows the client's BSK public key which, if the client does not gratuitously expose its public key, can be used to obtain a modicum of correctness, that the client is connecting to the correct network (see [duckling]).

## Alignment with Wi-Fi Alliance Device Provisioning Profile

The definition of the BSK public key aligns with that given in [DPP]. This, for example, enables the QR code format as defined in [DPP] to be reused for TLS-POK. Therefore, a device that supports both wired LAN and Wi-Fi LAN connections can have a single QR code printed on its label, or dynamically display a single QR code on a display, and the bootstrap key can be used for DPP if the device bootstraps against a Wi-Fi network, or TLS-POK if the device bootstraps against a wired network. Similarly, a common bootstrap public key format could be imported into a BOM into a server that handles devices connecting over both wired and Wi-Fi networks.

Any bootstrapping method defined for, or used by, [DPP] is compatible with TLS-POK.

# Bootstrapping in TLS 1.3

Bootstrapping in TLS 1.3 leverages Certificate-Based Authentication with an External Pre-Shared Key {{!RFC8773}}. The External PSK (EPSK) is derived from the BSK public key, and the EPSK is imported using {{!RFC9258}}. This BSK MUST be from a cryptosystem suitable for doing ECDSA.

The TLS PSK handshake gives the client proof that the server knows the BSK public key. Certificate based authentication of the client by the server is carried out using the BSK, giving the server proof that the client knows the BSK private key. This satisfies the proof of ownership requirements outlined in {{introduction}}.

## External PSK Derivation

An {{RFC9258}} EPSK is made up of the tuple of (Base Key, External Identity, Hash). The EPSK is derived from the BSK public key using {{!RFC5869}} with the hash algorithm from the ciphersuite:

~~~
epsk   = HKDF-Expand(HKDF-Extract(<>, bskey),
                       "tls13-imported-bsk", L)
epskid = HKDF-Expand(HKDF-Extract(<>, bskey),
                       "tls13-bspsk-identity", L)
where:
  - epsk is the EPSK Base Key
  - epskid is the EPSK External Identity
  - <> is a NULL salt 
  - bskey is the DER-encoded ASN.1 subjectPublicKeyInfo
    representation of the BSK public key
  - L is the length of the digest of the underlying hash
    algorithm 
~~~

The {{!RFC9258}} ImportedIdentity structure is defined as:

~~~
struct {
   opaque external_identity<1...2^16-1>;
   opaque context<0..2^16-1>;
   uint16 target_protocol;
   uint16 target_kdf;
} ImportedIdentity;
~~~

and is created using the following values:

~~~
external_identity = epskid
context = "tls13-bsk"
target_protocol = TLS1.3(0x0304)
target_kdf = HKDF_SHA256(0x0001)
~~~

The EPSK and ImportedIdentity are used in the TLS handshake as specified in {{!RFC9258}}.

A performance versus storage tradeoff a server can choose is to precompute the identity of every bootstrapped key with every hash algorithm that it uses in TLS and use that to quickly lookup the bootstrap key and generate the PSK. Servers that choose not to employ this optimization will have to do a runtime check with every bootstrap key it holds against the identity the client provides.

## TLS 1.3 Handshake Details

The client includes the "tls_cert_with_extern_psk" extension in the ClientHello, per {{!RFC8773}}. The client identifies the BSK by inserting the serialized content of ImportedIdentity into the PskIdentity.identity in the PSK extension, per {{!RFC9258}}. The server looks up the client's EPSK key in its database using the mechanisms documented in {{!RFC9258}}.  If no match is found, the server SHALL terminate the TLS handshake with an alert.

If the server found the matching BSK, it includes the "tls_cert_with_extern_psk" extension in the ServerHello message, and the corresponding EPSK identity in the "pre_shared_key" extension. When these extensions have been successfully negotiated, the TLS 1.3 key schedule SHALL include both the EPSK in the Early Secret derivation and an (EC)DHE shared secret value in the Handshake Secret derivation. 

After successful negotiation of these extensions, the full TLS 1.3 handshake is performed with the additional caveat that the client authenticates with a raw public key (its BSK) per {{!RFC7250}}. The BSK is always an elliptic curve key pair, therefore the ClientCertTypeExtension SHALL always indicate RawPublicKey and the type of the client's Certificate SHALL be ECDSA and contain the client's BSK public key as a DER-encoded ASN.1 subjectPublicKeyInfo SEQUENCE.

Note that the client does not share its BSK public key with the server until after the client has completed processing of the ServerHello and verified the TLS key schedule. The PSK proof has completed at this stage, and the server has proven to the client that is knows the BSK public key, and it is therefore safe for the client to send the BSK public key to the server in the Certificate message. If the PSK verification step fails when processing the ServerHello, the client terminates the TLS handshake and the BSK public key is not shared with the server.

When the server processes the client's Certificate it MUST ensure that it is identical to the BSK public key that it used to generate the EPSK and ImportedIdentity for this handshake. 

When clients use the [duckling] form of authentication, they MAY forgo the checking of the server's certificate in the CertificateVerify and rely on the integrity of the bootstrapping method employed to distribute its key in order to validate trust in the authenticated TLS connection. 

The handshake is shown in Figure 1.

~~~

         Client                                            Server
         --------                                          --------
         ClientHello
         + cert_with_extern_psk
         + client_cert_type=RawPublicKey
         + key_share
         + pre_shared_key           -------->
                                                        ServerHello
                                             + cert_with_extern_psk
                                    + client_cert_type=RawPublicKey
                                                        + key_share
                                                   + pre_shared_key
                                              {EncryptedExtensions}
                                               {CertificateRequest}
                                                      {Certificate}
                                                {CertificateVerify}
                                    <--------            {Finished}
         {Certificate}
         {CertificateVerify}
         {Finished}                 -------->
         [Application Data]         <------->    [Application Data]
~~~

                    Figure 1: TLS 1.3 TLS-POK Handshake

# Using TLS Bootstrapping in EAP

Upon "link up", an Authenticator on an 802.1X-protected port will issue an EAP Identity request to the newly connected peer. For unprovisioned devices that desire to take advantage of TLS-POK, there is no initial realm in which to construct an NAI (see {{?RFC4282}}) so the initial EAP Identity response SHOULD contain simply the name "tls-pok@eap-dpp.arpa" in order to indicate to the Authenticator that an EAP method that supports TLS-POK SHOULD be started.

~~~
   Authenticating Peer     Authenticator
   -------------------     -------------
                            <- EAP-Request/
                            Identity

    EAP-Response/
    Identity
    (tls-pok@eap-dpp.arpa) ->

                            <- EAP-Request/
                            EAP-Type=TEAP
                           (TLS Start)

    EAP-Response/
    EAP-Type=TEAP
    (TLS client_hello with
     tls_cert_with_extern_psk
     and pre_shared_key) ->
     
                       .
                       .
                       .
~~~

Both client and server have derived the EPSK and associated {{!RFC9258}} ImportedIdentity from the BSK as described in {{external-psk-derivation}}. When the client starts the TLS exchange in the EAP transaction, it includes the ImportedIdentity structure in the pre_shared_key extension in the ClientHello. When the server received the ClientHello, it extracts the ImportedIdentity and looks up the EPSK and BSK public key. As previously mentioned in {{bootstrap-key-pair}}, the exact mechanism by which the server has gained knowledge of or been provisioned with the BSK public key is outside the scope of this document.

The server continues with the TLS handshake and uses the EPSK to prove that it knows the BSK public key. When the client replies with its Certificate, CertificateVerify and Finished messages, the server MUST ensure that the public key in the Certificate message matches the BSK public key.

Once the TLS handshake completes, the client and server have established mutual trust. The server can then proceed to provision a credential onto the client using, for example, the mechanisms outlined in {{?RFC7170}}.

The client can then use this provisioned credential for subsequent network authentication. The BSK is only used during bootstrap, and it not used for any subsequent network access.

# IANA Considerations

This document registers the following arpa domain:

eap-dpp.arpa

# Security Considerations 

Bootstrap and trust establishment by the TLS server is based on proof of knowledge of the client's bootstrap public key, a non-public datum. The TLS server obtains proof that the client knows its bootstrap public key and, in addition, also possesses its corresponding private key.

Trust on the part of the client is based on successful completion of the TLS 1.3 handshake using the EPSK derived from the BSK. This proves to the client that the server knows its BSK public key. In addition, the client assumes that knowledge of its BSK public key is not widely disseminated and therefore any server that proves knowledge of its BSK public key is the appropriate server from which to receive provisioning, for instance via {{?RFC7170}}. [duckling] describes a security model for this type of "imprinting".

An attack on the bootstrapping method which substitutes the public key of a corrupted device for the public key of an honest device can result in the TLS sever on-boarding and trusting the corrupted device.

If an adversary has knowledge of the bootstrap public key, the adversary may be able to make the client bootstrap against the adversary's network. For example, if an adversary intercepts and scans QR labels on clients, and the adversary can force the client to connect to its server, then the adversary can complete the TLS-POK handshake with the client and the client will connect to the adversary's server. Since physical possession implies ownership, there is nothing to prevent a stolen device from being on-boarded. 
