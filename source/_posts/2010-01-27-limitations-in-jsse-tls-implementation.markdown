---
layout: post
title: "Limitations in JSSE TLS implementation"
date: 2010-01-27 21:59:30 +0100
comments: true
categories: [Java, TLS]

---

In the context of the german electronic health card (EHC) the specifications allow for TLS connections with high need for protection the following SSL cipher suites only:

* TLS_DHE_RSA_WITH_AES_128_CBC_SHA
* TLS_DHE_RSA_WITH_AES_256_CBC_SHA
* TLS_DHE_RSA_WITH_3DES_EDE_CBC_SHA

Furthermore for the Diffie-Hellman KeyExchange usage of Group 5 with a Modulus of 1536 Bit is required.
Implementing the TLS client side using standard Java one is running into limitations of the current JSSE. The first problem is the Diffie-Hellman KeyPairGenerator from Sun. It has a hardcoded limitation to a modulus <= 1024 Bit:

``` java Length check in DH KeyPairGenerator
if ((pSize < 512) || (pSize > 1024) || (pSize % 64 != 0)) {
    throw new InvalidAlgorithmParameterException("Prime size must be multiple of 64, and can only range "
    + "from 512 to 1024 (inclusive)");
}

```
A solution for this problem is to replace the Sun Diffie-Hellman KeyPairGenerator with the Bouncycastle implementation. 
It is sufficient to install the BC security provider with a higher priority than the default Sun Provider.
Having done this keypair generation runs fine now but another problem arises. The Sun TlsPrfGenerator class responsible for calculating the TLS master secret from the premaster secret throws an ArrayIndexOutOfBoundsException because it can not handle a premaster secret size >= 128 Bit. A solution might be to replace the TlsMasterSecretKeyGenerator (that internally calls TlsPrfGenerator functions) implementation with a fixed version. A custom implementation must be shipped as a signed security provider, requiring to order a Service Provider certificate from Sun. Alternatively one could use the TLS implementation from IAIK,
which works out of the box with a DH modulus length > 1024 Bit.
Bouncycastle also provides a lightweight TLS API, but at the moment it lacks TLS client authentication, 
which is a must-have in my scenario.




