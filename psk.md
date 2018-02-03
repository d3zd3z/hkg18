# Pre-shared keys

Although public key cryptography has many benefits that are useful for
authentication, often in IoT, the overhead proves to be too much, and
systems settle for something more simple, known as pre shared keys.

In this scenario, in order for the client and server to communicate,
they must communicate with a secret key that is know to both of them.
It is important that these protocols are designed such that either
successful or faulty authentication will not reveal anything about the
key.  There are also issues with offline attacks being done on the
key.

## Big keys

RFC 4279 defines a set of ciphersuites for TLS based on using pre
shared keys to mutually authenticate the client with the server.
These suites rely on these shared keys being large (similar in size to
the keys used in symmetric encryption), since smaller keys could be
brute forced by someone observing the protocol.

.. How this works

LWM2M, for example, defines a mode that uses these pre-shared keys.

## Smaller keys

RFC 8236 describes another ciphersuite, called J-PAKE, that uses what
is known as a zero-knowledge proof, allowing the keys used to be
smaller, without allowing them to be brute forced (this is the idea
behind zero knowledge) based on observing the traffic.  These
protocols are very new (the RFC was accepted Sept of 2017), and,
although some potential weaknesses have been found, these can be
mitigated by careful use of the protocols.  The smaller keys allow
mutual authentication with a smaller key, such as a user-entered PIN.

The Thread protocol makes use of J-PAKE.
