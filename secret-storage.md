# Secret storage from threat modeling

- Been doing threat modeling
- Common theme is the storage of secrets
- Today I want to talk about these secrets.

- [Normal](normal.md)
- [PSK](psk.md)

## The problem

- Security requires mutual authentication.
- PSK is very common in IoT, due to fewer resources needed.
- Generally, some key has to be installed in the device at
  provisioning time.
- This "network key" can result in more of a class break, as devices
  could pretend to be real network devices.
- A vulnerability anywhere in the code that can allow reads from
  memory can reveal these keys to an attacker.  May even happen over a
  network interface.

## Other solutions

### TrustZone(tm)

- Larger CPUs.  Common in A CPUs.
- An example is OP-TEE.
- Doesn't really help us with M CPUs.

### TrustZone(tm) for V8m

- Arm's solution for V8m CPUs
- Allows a trusted environment to separate secret data
- But, only available on V8m cores

### Privilege Separation

- Traditional, Unix style, for example.

- Really wants an MMU, and even then is heavy for systems with small
  memory.
- Lots of code on the wrong side of the barrier.  Think network
  stacks, etc.  Doesn't satisfy least privilege.

## What can we do?

### V8m

- Tell everyone to only use V8m.
- This may work at some point, but V(<8)m will be around for quite a
  while, cost.

### Privilege Separation improvements

- Move more code into "user" side.
- Still has problem with really wanting an MMU and more RAM.

### Separate CPU

- Do security on a separate MCU
- Just for secrets, or network stuff
- What about other code on this MCU

### Hypervisorish Thing

- mbed/uVisor, but heavy with MPU
- Simplified version possible for us?
