# Normal TLS

## Revamp

- Symmetric and Asymmetric encryption
  how symmetric encryption works
  how asymmetric works, roughtly
- Very basics of RSA
  Illustration? with layers is ok to show as well
  [Signature](signature.svg)
- What does it mean to sign
  Animation: document
- What a certificate is: some data that is signed
- Self signed certificates

## Introduction

- Before getting into IoT security, let's take a little time to cover
  how regular SSL/TLS works.

  The most common use of SSL/TLS that most of us use regularly is done
  within our web browser.

     browser example here.

- Start with public/private key.
- What are certificates

- In order for this to work, we need to have what are known as CAs
  (certificate authorities).  These are entities that we as users of
  the web trust (or at least have to trust) to authenticate the sites
  that we visit.

  There are a lot of them, I just checked, and my machine has 161 of
  these.  Firefox includes 162.

- Let's say I want to have TLS work with davidb.org.  I use
  letsencrypt, and through a process where I convince them that I can
  control davidb.org, they issue me a certificate for this domain.

  The certificate they issue me has some information in it:

  $ openssl x509 -in davidb.org-cert.pem -inform PEM -noout -text

    Signature Algorithm: sha256WithRSAEncryption
        Issuer: C=US, O=Let's Encrypt, CN=Let's Encrypt Authority X3
        Validity
            Not Before: Jan 16 23:22:39 2018 GMT
            Not After : Apr 16 23:22:39 2018 GMT
        Subject: CN=davidb.org
        Subject Public Key Info:
	   RSA public info
        X509v3 extensions:
	    ...
            X509v3 Subject Alternative Name:
                DNS:davidb.ninja, DNS:davidb.org, DNS:fossil.davidb.org, DNS:www.davidb.ninja, DNS:www.davidb.org

  For us, the important things here are, the validity, which tells us
  when this cert is valid. The Subject as well as the Subject
  Alternative Name tells the recipient what web addresses this
  certificate is for.  The certificate also contains the "Subject
  Public Key Info".  What they are certifying is that this public key
  is associated with these web pages.

  This whole package is then signed by Let's Encrypt, and can be
  verified using their public key, which is one of the 162 keys that
  are baked into Firefox.

- So what happens when I try to connect to davidb.org:

  $ openssl s_client -connect davidb.org:443
  ...
  Press ^D to exit here

  This will spew a bunch of information out, including the contents of
  the certificate we analyzed just a few minutes ago.

  There are several things that happen with a TLS connection.  Part of
  this is about negotiating what versions of the protocol they two
  parties will use, and specifically, they negotiate a specific
  cyphersuite.  In my case, it is ECDHE-RSA-AES128-GCM-SHA256, which
  tells us a bunch of things:

    - ECDHE is the key exchange algorithm (Elliptic curve
      Diffie-Hellman Ephemeral).  This algorithm allows both sides
      of the protocol to agree on a session key without an observer
      being able to tell what that is.  To prevent party-in-the-middle
      attacks, the communication will be signed with the RSA key
      associated with the given certificate.

      One thing important here is that this key exchange is done with
      an ephemeral (randomly generated) key.  This gives it a property
      known as perfect forward secrecy.  This means if the servers
      private key is ever compromised or revealed, that information
      cannot be used to decrypt this particular communication.

      The exchange, however, is signed with the server's private key.
      Since the client trusts the public key in the certificate (in as
      much as they trust the CA system), this gives the client an
      assurance that they are communicating with this server.

      On the other hand, this tells the server nothing about the
      client.  The server can't really even know whether there is an
      intercepting party in the communication.  Presumably, the
      clients software would reject the connection if it detected
      this, but the server can't trust the client's software.

      This is why your bank's website needs you to log in.  It is
      possible for the client to also present a certificate that
      allows the communication to be mutually authenticated.  However
      managing client certificates has shown to be overly complicated
      for a majority of web uses.  There are specialized applications
      where this is indeed done.

## Big picture

So, looking back at this, what kind of information needs to be
protected in this scenario.

  - The client doesn't really have any secrets.  To really
    authenticate, it will need to present a password, but this isn't
    really part of the TLS protocol.

  - The server has a private key for the keypair in the server
    certificate.  Anyone who gets ahold of this private key can
    impersonate the server.

## Points of failure

  - Must trust the CAs, all of them.  Since the browsers will trust a
    certificate signed by any of the CAs, a single rogue CA can make
    certificates for arbitrary sites.  Classically, this happened with
    a compromised CA that created a cert for *.google.com.
    
    Several solutions.  This specific CA (*.google.com) is blacklisted
    by browsers.  We also have the CAA policy where sites can
    advertise, out of band (through DNS) which CAs are authorized to
    sign for those domains.  Browser are also starting to bake this
    information in for high profile sites.

  - Will the user notice if the padlock icon isn't present?  Browser
    are getting better about refusing to show insecure webpages.

  - TLS is complex, there are likely still bugs in the implementation.

  - Ciphersuite negotiation can result in a weak cipher suite.  Again,
    the browser are getting better at just refusing to work with sites
    that aren't configured securely.  Support for PFS is coming
    slowly, and browser don't generally reject this.

## From IoT

  - Need client certs to make this work, since IoT devices aren't
    really interactive and able to "log in" to a form.

  - The regular CA infrastructure is large (even if stored compactly,
    the 162 certs in Firefox would be about 180K, which is larger than
    the flash space available on many IoT devices.  Even large devices
    (with say 1MB of flash) would have a hard time justifying this
    much space to the root CA list.

    This list could certainly be pruned for the IoT device, to just
    the CAs that are expected to be used by the service.

    It is also possible to not use the existing CA infrastructure at
    all.  Since the IoT devices and services are generally managed
    together, the company providing this can act as their own CA,
    putting their own root certs on the devices (and server) and
    managing all of the certificates themselves.

    It is also likely that some of the existing CA companies will step
    in to provide this kind of service.  It will likely be expensive
    (they are going to be interested in making money, after all), but
    they can probably do this job more securely than a company with no
    experience in it.
