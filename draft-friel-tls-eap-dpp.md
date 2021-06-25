---

title: "Bootstrapped TLS Authentication"
abbrev: TLS-POK
docname: draft-friel-tls-eap-dpp-02
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
        ins: R. Anderson
        name: Ross Anderson

    title: "The Ressurecting Ducking: Security Issues for Ad-Hoc Wireless Networks"
    date: 1999

--- abstract

This document defines a TLS extension that enables a server to prove to a client that it has knowledge of the public key of a key pair where the client has knowledge of the private key of the key pair. Unlike standard TLS key exchanges, the public key is never exchanged in TLS protocol messages. Proof of knowledge of the public key is used by the client to bootstrap trust in the server. The use case outlined in this document is to establish trust in an EAP server.

--- middle


# Introduction

On-boarding of devices with no, or limited, user interface can be difficult.  Typically, a credential is needed to access the network
and network connectivity is needed to obtain a credential.  This poses a catch-22.

If trust in the integrity of a device's public key can be obtained in an out-of-band fashion, a device can be authenticated and provisioned with a usable credential for network access.  While this authentication can be strong, the device's authentication of the network is somewhat weaker.  [duckling]] presents a functional security model to address this asymmetry.

There are on-boarding protocols, such as [DPP], to address this use case but they have drawbacks. [DPP] for instance does not support wired network access.  This document describes an on-boarding protocol, which we refer to as TLS Proof of Knowledge or TLS-POK.

## Bootstrap Key Pair

The mechanism for on-boarding of devices defined in this document relies on bootstrap key pairs. A client device has an associated elliptic curve (EC) key pair. The key pair may be static and baked into device firmware at manufacturing time, or may be dynamic and generated at on-boarding time by the device. If this public key, specifically the ASN.1 SEQUENCE SubjectPublicKeyInfo from {{?RFC5280}}, can be shared in a trustworthy manner with a TLS server, a form of "origin entity authentication" (the step from which all subsequent authentication proceeds) can be obtained. 

The exact mechanism by which the server gains knowledge of the public key is out of scope of this specification, but possible mechanisms include scanning a QR code to obtain a base64 encoding of the ASN.1-formatted public key or upload of a Bill of Materials (BOM). If the QR code is physically attached to the client device, or the BOM is associated with the device, the assumption is that the public key obtained in this bootstrapping method belongs to the client. In this model, physical possession of the device implies legitimate ownership.

The server may have knowledge of multiple bootstrap public keys corresponding to multiple devices, and TLS extensions are defined in this document that enable the server to identity a specific bootstrap public key correspinding to a specific device.

Using the process defined herein, the client proves to the server that it has possession of the private analog to its public bootstrapping key. Provided that the mechanism in which the server obtained the bootstrapping key is trustworthy, a commensurate amount of authenticity of the resulting connection can be obtained. The server also proves that it knows the client's public key which, if the client does not gratuitously expose its public key, can be used to obtain a modicum of correctness, that the client is connecting to the correct network (see [duckling]).

## Alignment with Wi-Fi Alliance Device Provisioning Profile

The definition of the boostrap public key aligns with that given in [DPP]. This, for example, enables the QR code format as defined in [DPP] to be reused for TLS-POK. Therefore, a device that supports both wired LAN and Wi-Fi LAN connections can have a single QR code printed on its label, and the bootstrap key can be used for DPP if the device bootstraps against a Wi-Fi network, or TLS-POK if the device bootstraps against a wired network. Similarly, a common bootstrap public key format could be imported in a BOM into a server that handles devices connecting over both wired and Wi-Fi networks.

Any bootstrapping method defined for, or used by, [DPP] is compatible with TLS-POK.

# Bootstrapping in TLS 1.3

The bootstrapping modifications introduce an extension to identify a "bootstrapping" key which is converted into an external PSK and used directly in the TLS 1.3 handshake.  This key MUST be from a cryptosystem suitable for doing ECDSA. 

## Bootstrap Extended PSK

This document defines the "bskey" extended PSK type by expanding on the work in {{?I-D.ietf-group-tls-extensible-psks}}.

     enum {
       bskey(TBD), (255)
     } ExtendedPskIdentityType;

A bskey PSK is a varient of an external PSK which is derived from a static source, in this case a public key. 

The PSKIdentity of a bskey extended PSK is encoded with a string derived from the DER-encoded ASN.1 subjectPublicKeyInfo representation of the bootstrapping public key.

     struct {
       opaque identity<1..2^32-1>
     } BootstrapPSKIdentity;


Both the bskey PSK and the BootstrapPSKIdentity are computed using {{!RFC5869}}:

~~~
bskeypsk = HKDF-Expand(HKDF-Extract(<>, bskey),
                       "tls13-extended-psk-bskey", L)
identity = HKDF-Expand(HKDF-Extract(<>, bskey),
                       "tls13-psk-identity-bskey", L)
where:
  - <> is a NULL salt 
  - bskey is the DER-encoded ASN.1 subjectPublicKeyInfo
    representation of the bootstrapping key
  - L is the length of the underlying hash algorithm used in the
    ciphersuite
~~~

[[ Open question: should we use HKDF w/SHA-256 for the identity so the server can store digests along with the bootstrapping keys and not require the server to generate them on the fly with a possibly different hash algorithm? It makes sense to have the bskeypsk derivation use the underlying hash algorithm because the resulting PSK is going to be inserted into the TLS 1.3 key schedule which will use HKDF with that hash algorithm and having the PSK be of suitable size is A Good Thing, but we might want to optimize server performance by hard-wiring SHA-256 for the identity. Discuss! ]]


##  Changes to TLS 1.3 Handshake

The client includes the "tls_cert_with_extern_psk" extension in the ClientHello, per {{!RFC8773}}, and identifies the bootstrapping key using the BootstrapPSKIdentity extension. The server looks up the client's bootstrapping key in its database by checking the hash of each entry with the value received in the ClientHello.  If no match is found, the server SHALL terminate the TLS handshake with an alert.

[[ TODO: should we define an explicit unknown_bsk_identity alert, similar to unknown_psk_identity ]]

If the server found the matching bootstrap key, it generates the bskeypsk and includes the "tls_cert_with_extern_psk" extension in the ServerHello message. When these extensions have been successfully negotiated, the TLS 1.3 key schedule SHALL include both the bskeypsk in the Early Secret derivation and an (EC)DHE shared secret value in the Handshake Secret derivation. 

After successful negotiation of these extensions, the full TLS 1.3 handshake is performed with the additional caveat that the client authenticates with a raw public key (its bootstrapping key) per {{!RFC7250}}. The bootstrapping key is always an elliptic curve public key, therefore the ClientCertTypeExtension SHALL not be included in the ClientHello as it is implied by the presence, and acceptance, of the bootstrapping key identifer. The type of the client's Certificate SHALL be RawPublicKey and contain the client's bootstrapping key as a DER-encoded ASN.1 subjectPublicKeyInfo SEQUENCE.

When the server processes the client's Certificate it MUST ensure that it is identical to the bootstrapping public key that it used to generate an external PSK and PSKIdentifier for this handshake. 

The handshake is shown in Figure 1.

~~~

         Client                                            Server
         --------                                          --------
         ClientHello
         + bskey_id
         + cert_with_extern_psk
         + key_share                -------->
                                                        ServerHello
                                                         + bskey_id
                                             + cert_with_extern_psk
                                                        + key_share
                                              {EncryptedExtensions}
                                                {CertificatRequest}
                                                      {Certificate}
                                    <--------            {Finished}
         {Certificate}
         {Finished}                 -------->
         [Application Data]         <------->    [Application Data]
~~~


                    Figure 1: TLS 1.3 TLS-POK Handshake

# Using TLS Bootstrapping in EAP

Enterprise deployments typically require an 802.1X/EAP-based authentication to obtain network access. Protocols like {{?RFC7030}} can be used to enroll devices into a Certification Authority to allow them to authenticate using 802.1X/EAP. But this creates a Catch-22 where a certificate is needed for network access and network access is needed to obtain certificate.

Devices whose bootstrapping key can been obtained in an out-of-band fashion can perform an EAP-TLS-based exchange, for instance {{?RFC7170}}, and authenticate the TLS exchange using the bootstrapping extensions defined in {{bootstrapping-in-tls-13}}. This network connectivity can then be used to perform an enrollment protocol (such as provided by {{?RFC7170}}) to obtain a credential for subsequent network connectivity and certificate lifecycle maintenance.

Upon "link up", an Authenticator on an 802.1X-protected port will issue an EAP Identify request to the newly connected peer. For unprovisioned devices that desire to take advantage of TLS-POK, there is no initial realm in which to construct an NAI (see {{?RFC4282}}) so the initial EAP Identity response SHOULD contain simply the name "TLS-POK" in order to indicate to the Authenticator that an EAP method that supports TLS-POK SHOULD be started.

~~~
	   Authenticating Peer     Authenticator
 	   -------------------     -------------
	                            <- EAP-Request/
				    Identity

	    EAP-Response/
	    Identity (TLS-POK) ->

	                            <- EAP-Request/
				    EAP-Type=TEAP
				    (TLS Start)
			       .
			       .
			       .
~~~


# Summary of Work

[TODO: agree with WG chairs where this work lives and where it should be documented.]

The protocol outlined here can be broadly broken up into 4 distinct areas:

- TLS extensions to transport the bootstrap public key identifier
- Use of the TLS 1.3 extension for certificate-based authentication with an external PSK
- The client's use of a raw public key in its certificate
- TEAP extensions to leverage the new TLS-POK handshake for trust establishment

This document captures all 4 areas, but it may be more appropriate to merge into an existing document.

# IANA Considerations

IANA will allocated an ExtensionPSKIdentityType for the bskey type from the TLS 1.3 repository created by {{?I-D.ietf-group-tls-extensible-psks}} and replace TBD in this document with that number.

# Security Considerations 

Bootstrap and trust establishment by the TLS server is based on proof of knowledge of the client's bootstrap public key, a non-public datum. The TLS server obtains proof that the client knows its bootstrap public key and, in addition, also possesses its corresponding private analog.

Trust on the part of the client is based on validation of the server certificate and the TLS 1.3 handshake. In addition, the client assumes that knowledge of its public bootstrapping key is not widely disseminated and therefore any device that proves knowledge of it is the appropriate device from which to receive provisioning, for instance via {{?RFC7170}}.

An attack on the bootstrapping method which substitutes the public key of a corrupted device for the public key of an honest device can result in the TLS sever on-boarding and trusting the corrupted device.

If an adversary has knowledge of the bootstrap public key, the adversary may be able to make the client bootstrap against the adversary's network. For example, if an adversary intercepts and scans QR labels on clients, and the adversary can force the client to connect to its server, then the adversary can complete the TLS-POK handshake with the client and the client will connect to the adversary's server. Since physical possession implies ownership, there is nothing to prevent a stolen device from being on-boarded. 

