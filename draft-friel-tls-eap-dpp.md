---

title: "Bootstrapped TLS Authentication"
abbrev: TLS-POK
docname: draft-friel-tls-eap-dpp-00
category: std

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
    date: 2019
  Stajano:
    author:
      name: Stajano, F. and R. Anderson
    title: "The Ressurecting Ducking: Security Issues for Ad-Hoc Wireless Networks"
    date: 1999

--- abstract

blah abstract

--- middle


# Introduction

On-boarding of devices with no, or limited, user interface can be difficult.  Typically, a credential is needed to access the network
and network connectivity is needed to obtain a credential.  This poses a catch-22.

If trust in the integrity of a device's public key can be obtained in an out-of-band fashion, a device can be authenticated and provisioned with a usable credential for network access.  While this authentication can be strong, the device's authentication of the network is somewhat weaker.  [Stajano]] presents a functional security model to address this asymmetry.

There are on-boarding protocols, such as [DPP], to address this use case but these do not address large-scale, enterprise deployments that require an X.509 certificate for network access.  This memo describes such a protocol.

# Using Bootstrapping in TLS 1.3

The bootstrapping modifications introduce a extension to identify a "bootstrapping" key into the TLS 1.3 handshake.  This key MUST be from a cryptosystem suitable for doing (EC)DH.  When using the bootstrapping extension, two (EC)DH operations are performed, a static-ephemeral (using the client's bootstrapping key and the server's ephemeral key) and an ephemeral-ephemeral (using the client's ephemeral key and the server's ephemeral key).

##  Changes to TLS 1.3 Handshake

The bootstrapping key is identified in the ClientHello using the "bskey" extension.  The contents of this extension are a SHA256 hash of a DER-encoded ASN.1 represention of the subjectPublicKeyInfo of device's public key.  The bskey extension is defined in Section 2.2.

The server looks up the client's bootstrapping key in its database by checking the SHA256 hash of each entry with the value received in the ClientHello.  If no match is found, the server SHOULD respond with a HelloRetryRequest indicating a desired "key_share" extension.  A client that has sent a ClientHello with the "bskey" extension that receives a HelloRetryRequest SHALL assume the server does not have its bootstrapping key.  It MAY decide to continue with another TLS ciphersuite.

The handshake is shown in Figure 1.

~~~

         Client                                            Server
         --------                                          --------
         ClientHello (bskey)
         + key_share                -------->
                                                        ServerHello
                                                        + key_share
                                              {EncryptedExtensions}
                                                         {Finished}
                                    <--------   [Application Data*]
         {Finished}                 -------->
         [Application Data]         <------->    [Application Data]
~~~


                    Figure 1: TLS 1.3 TLS-PWD Handshake

The server looks up the client's bootstrapping key based upon the received bskey extension.  The server generates an ephemeral ECDH keypair and performes two ECDH operations with it, one using the client's bootstrapping public key and another using the client's ECDH public key from the keyshare extension.  The x-coordinate of the two ECDH shared secret points are concatenated, bootstrapping key first, ephemeral key second, to create a concatenated shared secret for the TLS 1.3 key schedule.

The key schedule for bootstrapping TLS is as follows.

~~~
                                       0
                                       |
                                       v
                         PSK ->  HKDF-Extract = Early Secret
                                       |
                                       +-----> Derive-Secret(...)
                                       +-----> Derive-Secret(...)
                                       +-----> Derive-Secret(...)
                                       |
                                       v
                                 Derive-Secret(., "derived", "")
                                       |
                                       v
   concatenated_shared_secret -> HKDF-Extract = Handshake Secret
                                       |
                                       +-----> Derive-Secret(...)
                                       +-----> Derive-Secret(...)
                                       |
                                       v
                                 Derive-Secret(., "derived", "")
                                       |
                                       v
                            0 -> HKDF-Extract = Master Secret
                                       |
                                       +-----> Derive-Secret(...)
                                       +-----> Derive-Secret(...)
                                       +-----> Derive-Secret(...)
                                       +-----> Derive-Secret(...)
~~~


## Bootstrapping Key Naming

The bootstrapping key of the client is specified using the bskey extension.  The "extension data" field of this extension SHALL consist of the base64 encoded SHA256 digest of the DER-encoded ASN.1 subjectPublicKeyInfo representation of the bootstrapping public key. The extension is:

      opaque bskey[32];

The bskey is a 32 byte string.  The ExtensionType of bskey is TBD.

# Using TLS Bootstrapping in EAP

Enterprise deployments typically require an 802.1x/EAP-based authentication to obtain network access.  Devices whose boostrapping key has been obtained in an out-of-band fashion can perform an EAP-TLS-based exchange, for instance {{?RFC7170}}, and authenticate the TLS exchange using the bootstrapping extensions defined in Section 2.

# IANA Considerations

IANA will allocated an ExtensionType for the bskey extension from the appropriate TLS 1.3 repository and replace TBD in this document with that number.

# Security Considerations 

[[TODO]]


