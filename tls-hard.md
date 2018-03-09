# TLS and why it is hard

- Introduction

  Ideal magical world, TLS is a thing you drop into a connection and
  it becomes secure.  Both parties just communicate as if they were
  read/write send/recv packets, but everything is secured.

  The problem is that, although the communication is "secure", it is
  crucial that you know who you are talking to.  Without this, the
  encrypted part of the communication is completely meaningless.

- How does this work for my browser

  When I connect to `https://www.example.com/`, the server sends a
  certificate to me.  The things that are important about this
  certificate:

  - Either "Common Name (CN)" or "Certificate Subject Alt Name"
    contains the dns domain name of the site looked up.  So in this
    case, the certificate returned must either have CN of
    "www.example.com", or it must be one of the entries in the
    "Certificate Subject Alt Name".

  - The certificate validity period encompasses the current time.

  - The "Certificate Key Usage" and "Extended Key Usage" indicate this
    certificate is valid for "TLS Web Server Authentication", as well
    as "Signing" and "Key Encipherment".

  - There is a signature chain from this certificate to a root
    certificate that is in the browser's list of trusted root
    certificates (162 of these at my last check).

  - No certificate in the chain is on a black list of compromised
    certificates.

  - Other more modern techniques browsers use to only accept valid
    certificates, especially for popular domains.

  Once these things have been validated, and the rest of the TLS
  handshake is valid, the client and server will begin communicating
  over a secure and partially authenticated channel.

  - The client has reasonable assurance that the server is the correct
    site.

  - The server knows **nothing** about the client.  This is why many
    sites will then require you to log in.  Certificates can be used
    to authenticate the client, but this is uncommon because this
    would require users to manage certificates, and this would
    frequently be done insecurely.

- What does the browser do (TLS library) to do all of this:

  - Initialize the library, and create a new context.

  - Configure the context to verify the peer, by calling a callback
    function.

  - Configure options for TLS so that only modern, secure TLS features
    are used.

  - Load a database of root certificates.  For example:
    `/etc/ssl/certs/ca-certificates.crt`.

  - Setup the TLS library to use sockets.

  - Configure valid ciphers.

  - Tell the TLS library the hostname it thinks it is talking to.

  - TLS connect (does socket connection).

  - TLS handshake

  - Verify that the server presented a certificate.

  - Verify that the cert chain is trusted

  - Verify that the cert matches the host specified.

  - Whew, communicate with the server.

  The verify callback is optional and allows a particular connection
  to use different rules than the defaults to state that the chain is
  valid.

  There is a lot of stuff here, and it is easy to get wrong (often
  resulting in insecure connections).  We can optimize some of this
  (we shouldn't allow insecure ciphers, only use the latest TLS, and a
  lot is going to be common).

  Oh, and it isn't really quite that simple.

  - TLS allows for renegotiation.  Maybe this isn't needed to IoT
    usage, but it is part of the spec.

  - TLS/DTLS provide something known as session resumption.  Through
    storing some secret state about a particular communication
    channel, it can be reestablished later (between the same two
    parties) with less communication and computational effort.  Given
    the CPU constraints of IoT, session resumption is likely to be
    quite important.

- Bigger problems

  - This model of certificate validation doesn't usually work very
    well for IoT.

    - The default certificate chain is quite large (e.g. my current
      one is a 250KB file, which would take up 1/4 of the flash on my
      largest IoT SoC).

    - The default algorithms (typically RSA) can use large keys and
      require lots of RAM.

    - The default TLS record size is 16KB.  This can be smaller, but
      this is an optional TLS extension.  By default, the client will
      need just 32KB of RAM for buffers of TLS data.

    - This is all based on use over TCP.  Many IoT protocols are more
      suited to UDP.  The DTLS spec describes the protocol, but
      support is new, and the connections and usage are different in
      the use of the TLS library

    - Many IoT devices do not have a secure notion of time (may not
      even know the time).  The certificate validity periods become
      meaningless, and need to be either ignored, or this problem
      solved in another way.

  - Work on IoT protocols (specifically LWM2M and Thread) have
    specific solutions to this (there is also work on how to do this
    with MQTT).

    - LWM2M gives three options: use certs like the browser does, use
      PSK, use bare keys.

    - Thread specifies PSK using a specific (and different than LWM2M)
      ciphersuite.

    - These new features are only newly implemented in TLS libraries.

- Using these new approaches

  - PSK

    Pre-Shared Keys (PSK) provides a way for the client and server to
    mutually authenticate using a shared secret.  Getting and protecting
    this secret is a challenging task, and warrants its own study.

    Generally, these keys should be large randomly-generated numbers, to
    avoid brute-force attacks.  The JPAKE specified by Thread allows
    smaller keys to be used (typically a PIN entered by a user).

  - Certs

    Certificates in IoT will generally only allow a small number of
    root certs.

    Usually, the client will have to authenticate for this to be
    meaningful.  It is possible to do authentication at a higher level
    in the protocol, but this is easy to get wrong, and has little
    benefit.

    Distributing the private keys for these certs to the clients is a
    similar challenge to distributing PSK.

    Since Certs are more costly computationally than PSK, this is an
    uncommon approach for these IoT protocols.

  - Bare keys

    Bare keys can be used instead of certificates.  This requires both
    parties to keep track of the valid public key for every entity
    they communicate with.  The benefit is less complexity and
    computational work, but management of keys becomes more complex.

    TLS extensions to support Bare keys is more preliminary than
    either Certs or PSK.

- Handling this in a friendly TLS library:

  - We probably need to be able to handle all three approaches.
    Connections to public websites will be difficult, unless there is
    room to store the full cert chain.

  - We can define a context for security that supports these specific
    ways of doing things (or possibly just Certs, and both ways of
    doing PSK).

  - The setup will require giving enough information for the library
    to validate the other side of the connection, as well as to secure
    itself.

  - Since there are secrets involved, these will likely not be
    accessible to most code running on the CPU.  Secure APIs will need
    to be used and ways of referencing secrets stored across a secure
    API.

- The fundamental problem

  There is a strong argument that we shouldn't be doing any of this.

  People shouldn't be writing applications that communicate directly
  over TLS.  Although just connecting and authenticating is
  complicated, management of secrets (private keys) is also
  complicated, and very easy to get wrong.

  There is a lot of work going into LWM2M, MQTT, Thread, etc to make
  the protocols secure, and handle the multitude of management issues.
  Individuals that write applications directly with TLS will almost
  certainly get it wrong, resulting in only an illusion of security.

  My suggestion is that we work to get LWM2M, COAPS, Thread, MQTT, and
  possibly others to use security best practices, likely directly
  using something such as mbed TLS properly, and that we don't even
  advertise having direct TLS/DTLS support.
