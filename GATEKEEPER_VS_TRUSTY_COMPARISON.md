# Android Gatekeeper: AIDL Software vs Trusty TEE Implementation

## Overview

This document compares two different implementations of Android's Gatekeeper framework:

1. **AIDL Software Gatekeeper** - `platform/hardware/interfaces/gatekeeper/aidl/software`
2. **Trusty TEE Gatekeeper** - `platform/system/core/trusty`

Both implementations provide password verification and throttling services for Android, but they differ significantly in their architecture, security model, and deployment scenarios.

---

## 1. Architecture and Design

### AIDL Software Gatekeeper

**Location**: `platform/hardware/interfaces/gatekeeper/aidl/software`

**Architecture**:
- Pure software implementation running in Android userspace
- Uses AIDL (Android Interface Definition Language) for IPC communication
- Implements the `IGatekeeper` AIDL interface
- Runs as a regular Android HAL (Hardware Abstraction Layer) service
- No hardware-backed security features

**Components**:
```
├── IGatekeeper.aidl (Interface definition)
├── SoftGatekeeper.cpp (Main implementation)
└── Service integration with Android HAL
```

**Key Characteristics**:
- Straightforward C++ implementation
- Uses Android's standard crypto libraries
- Stores authentication tokens and keys in regular filesystem
- No TEE (Trusted Execution Environment) required
- Suitable for development, testing, and low-security scenarios

---

### Trusty TEE Gatekeeper

**Location**: `platform/system/core/trusty`

**Architecture**:
- Hardware-backed implementation using Trusted Execution Environment (TEE)
- Split architecture: Normal World (Android) + Secure World (Trusty OS)
- Communicates via Trusty IPC mechanism
- Critical operations execute in isolated secure environment
- Hardware-enforced security boundaries

**Components**:
```
Normal World (Android):
├── Trusty Gatekeeper HAL (Client side)
├── Trusty IPC communication layer
└── Integration with Android framework

Secure World (Trusty OS):
├── Gatekeeper Trusted Application (TA)
├── Secure storage for keys/secrets
├── Hardware-backed key derivation
└── Secure cryptographic operations
```

**Key Characteristics**:
- Two-world architecture (Normal + Secure)
- Hardware Root of Trust
- Isolated execution environment
- Protection against Android compromise
- Suitable for production devices requiring high security

---

## 2. Security Model

### AIDL Software Gatekeeper

**Security Level**: Software-only

**Threat Model**:
- ✅ Protects against casual attacks
- ✅ Provides basic password throttling
- ❌ **Vulnerable** if Android OS is compromised
- ❌ **Vulnerable** to root access attacks
- ❌ **No protection** against sophisticated attackers with system access
- ❌ Keys stored in software can be extracted

**Trust Boundary**:
- Trusts the entire Android OS
- Security depends on Android kernel and system integrity
- No hardware isolation

**Attack Resistance**:
- Low to Medium
- Sufficient for development/testing
- Not recommended for production devices with sensitive data

---

### Trusty TEE Gatekeeper

**Security Level**: Hardware-backed

**Threat Model**:
- ✅ Protects against OS-level compromises
- ✅ Resists root access attacks
- ✅ Hardware-enforced isolation
- ✅ Secure key storage immune to software attacks
- ✅ Protected against memory dumps and debugging
- ✅ Survives Android system compromises

**Trust Boundary**:
- Minimal trusted computing base (TCB)
- Trust only extends to:
  - Trusty OS microkernel
  - Hardware security features
  - Gatekeeper Trusted Application
- Android OS is **outside** the trust boundary

**Attack Resistance**:
- High
- Meets Android CDD (Compatibility Definition Document) requirements
- Required for devices with StrongBox support
- Suitable for production deployment

---

## 3. Implementation Details

### AIDL Software Gatekeeper

**Key Operations**:

1. **Enrollment** (`enroll`):
   - Accepts password
   - Generates salt using software PRNG
   - Computes password hash (PBKDF2 or scrypt)
   - Stores hash in regular filesystem
   - Returns authentication token

2. **Verification** (`verify`):
   - Accepts password and challenge
   - Retrieves stored hash from filesystem
   - Computes hash of provided password
   - Compares hashes
   - Implements software-based throttling
   - Returns authentication token if successful

3. **Key Derivation**:
   - Uses standard crypto libraries (OpenSSL/BoringSSL)
   - No hardware-backed key generation
   - Keys accessible to privileged processes

**Storage**:
- Uses Android's filesystem (`/data/vendor/gatekeeper/` or similar)
- Protected by standard Linux file permissions
- No hardware-backed secure storage

**Performance**:
- Fast (no context switches to secure world)
- Low latency
- Minimal overhead

---

### Trusty TEE Gatekeeper

**Key Operations**:

1. **Enrollment** (`enroll`):
   - Password sent to Trusty via secure IPC
   - Trusty generates salt using HWRNG (Hardware Random Number Generator)
   - Password hash computed in secure world
   - Hash stored in secure storage (RPMB or similar)
   - Authentication token generated with hardware-backed key
   - Token returned to Android

2. **Verification** (`verify`):
   - Password and challenge sent to Trusty via secure IPC
   - Trusty retrieves hash from secure storage
   - Hash computation in isolated environment
   - Hardware-enforced throttling (cannot be bypassed)
   - Authentication token signed with hardware key
   - Token returned to Android

3. **Key Derivation**:
   - Uses hardware-backed key derivation
   - Keys never leave secure world
   - Unique device-specific keys
   - Protected by hardware security features

**Storage**:
- RPMB (Replay Protected Memory Block) on eMMC
- Or secure filesystem provided by Trusty
- Hardware-enforced access control
- Data encrypted with hardware keys

**Performance**:
- Moderate overhead (context switches to TEE)
- Higher latency than pure software
- Acceptable for authentication use cases

**IPC Flow**:
```
Android Framework
       ↓
Gatekeeper HAL (Normal World)
       ↓ (Trusty IPC)
Secure Monitor / ARM TrustZone
       ↓
Trusty OS Kernel
       ↓
Gatekeeper TA (Secure World)
       ↓
Secure Storage / Hardware Crypto
```

---

## 4. Communication Mechanisms

### AIDL Software Gatekeeper

**Interface**: AIDL (Android Interface Definition Language)

**Characteristics**:
- Modern Android IPC using Binder
- Synchronous request-response model
- Standard Android HAL service registration
- No special privileges required beyond HAL access

**Advantages**:
- Well-documented and standardized
- Easy to implement and debug
- Compatible with Android's service model
- Good tooling support (AIDL compiler, dumpsys, etc.)

---

### Trusty TEE Gatekeeper

**Interface**: Trusty IPC

**Characteristics**:
- Custom IPC mechanism for TEE communication
- Message-based protocol
- Shared memory for large data transfers
- Special kernel driver required (`/dev/trusty-ipc`)

**Layers**:
1. **Android side**: Trusty userspace library (`libtrusty`)
2. **Kernel**: Trusty kernel driver
3. **Secure Monitor**: ARM SMC (Secure Monitor Call) or equivalent
4. **Trusty OS**: IPC message dispatcher

**Advantages**:
- Direct communication with secure world
- Efficient for security-critical operations
- Hardware-enforced isolation

**Complexity**:
- More complex than standard AIDL
- Requires platform-specific support
- Debugging is more difficult

---

## 5. Use Cases and Deployment Scenarios

### AIDL Software Gatekeeper

**Ideal For**:
- ✅ Development and testing environments
- ✅ Android Emulator (QEMU, Cuttlefish)
- ✅ Prototyping new features
- ✅ Low-security applications
- ✅ Educational purposes
- ✅ Budget devices without TEE hardware

**Not Suitable For**:
- ❌ Production devices with sensitive data
- ❌ Devices requiring Android CDD compliance for security
- ❌ Enterprise devices
- ❌ Devices requiring StrongBox Keymaster
- ❌ Devices processing payment information

**Example Devices**:
- Android Studio Emulator
- Cuttlefish virtual device
- Low-cost IoT devices without security requirements

---

### Trusty TEE Gatekeeper

**Ideal For**:
- ✅ Production smartphones and tablets
- ✅ Enterprise devices
- ✅ Devices with sensitive personal data
- ✅ Payment-capable devices
- ✅ Devices requiring Android CDD security compliance
- ✅ Devices with StrongBox Keymaster support

**Requirements**:
- Hardware TEE support (ARM TrustZone, Intel SGX, etc.)
- Trusty OS ported to platform
- Secure storage (RPMB or equivalent)
- Hardware crypto accelerators (recommended)

**Example Devices**:
- Google Pixel phones
- Many flagship Android smartphones
- Secure IoT gateways
- Industrial Android devices

---

## 6. Advantages and Disadvantages

### AIDL Software Gatekeeper

**Advantages**:
- ✅ **Simple**: Easy to implement and understand
- ✅ **Portable**: Works on any Android platform
- ✅ **Fast**: No TEE context switches
- ✅ **Debuggable**: Standard debugging tools work
- ✅ **Low barrier**: No special hardware required
- ✅ **Compatible**: Works with Android Emulator

**Disadvantages**:
- ❌ **Insecure**: Vulnerable to OS compromise
- ❌ **No isolation**: Keys accessible to root
- ❌ **Software-only**: No hardware root of trust
- ❌ **Non-compliant**: Doesn't meet Android CDD security requirements
- ❌ **Extractable**: Secrets can be dumped from memory/storage
- ❌ **Production risk**: Not suitable for real-world deployment

---

### Trusty TEE Gatekeeper

**Advantages**:
- ✅ **Secure**: Hardware-backed isolation
- ✅ **Compliant**: Meets Android CDD requirements
- ✅ **Protected**: Resists OS-level attacks
- ✅ **Certified**: Can be Common Criteria certified
- ✅ **Trustworthy**: Hardware root of trust
- ✅ **Production-ready**: Industry standard for secure devices

**Disadvantages**:
- ❌ **Complex**: Requires TEE platform support
- ❌ **Platform-specific**: Must be ported to each SoC
- ❌ **Slower**: Context switches add latency
- ❌ **Harder to debug**: Secure world debugging is limited
- ❌ **Hardware dependency**: Requires specific security hardware
- ❌ **Development cost**: More expensive to implement and maintain

---

## 7. Code Structure Comparison

### AIDL Software Gatekeeper Structure

```
hardware/interfaces/gatekeeper/aidl/
├── aidl/
│   └── android/hardware/gatekeeper/
│       ├── IGatekeeper.aidl          # Interface definition
│       ├── GatekeeperEnrollRequest.aidl
│       ├── GatekeeperEnrollResponse.aidl
│       ├── GatekeeperVerifyRequest.aidl
│       └── GatekeeperVerifyResponse.aidl
└── software/
    ├── SoftGatekeeper.cpp            # Main implementation
    ├── SoftGatekeeper.h
    ├── service.cpp                   # HAL service entry point
    └── Android.bp                    # Build configuration
```

**Implementation Style**:
- Monolithic C++ class
- Direct crypto library calls
- Simple file I/O for persistence
- Standard Android logging

---

### Trusty TEE Gatekeeper Structure

```
system/core/trusty/
├── gatekeeper/
│   ├── trusty_gatekeeper.cpp         # Android-side HAL
│   ├── trusty_gatekeeper.h
│   ├── trusty_gatekeeper_ipc.c       # IPC layer
│   └── trusty_gatekeeper_ipc.h
├── libtrusty/                         # Trusty IPC library
└── storage/                           # Secure storage interface

trusty/app/gatekeeper/                 # (Separate Trusty repository)
├── gatekeeper_ta.c                    # Trusted Application
├── manifest.c                         # TA manifest
├── secure_storage.c                   # Secure storage operations
└── rules.mk                           # Build configuration
```

**Implementation Style**:
- Split between Android HAL and Trusty TA
- Message serialization for IPC
- Secure world crypto operations
- Trusty-specific logging and error handling

---

## 8. Key Technical Differences Summary

| Aspect | AIDL Software Gatekeeper | Trusty TEE Gatekeeper |
|--------|-------------------------|----------------------|
| **Execution Environment** | Android userspace | Trusty OS (Secure World) |
| **IPC Mechanism** | AIDL/Binder | Trusty IPC |
| **Key Storage** | Filesystem | Secure storage (RPMB) |
| **Hardware Backing** | None | Yes (TEE) |
| **Attack Resistance** | Low | High |
| **Root Access Protection** | No | Yes |
| **Android CDD Compliant** | No | Yes |
| **Implementation Complexity** | Low | High |
| **Performance** | Fast | Moderate |
| **Debugging** | Easy | Difficult |
| **Production Suitability** | No | Yes |
| **Hardware Requirements** | None | TEE hardware |

---

## 9. Migration Path

### From Software to Trusty

For a device manufacturer moving from software gatekeeper to Trusty:

**Steps**:
1. **Hardware enablement**:
   - Ensure SoC supports ARM TrustZone or equivalent
   - Verify secure storage availability (RPMB)

2. **Trusty OS integration**:
   - Port Trusty OS to platform
   - Integrate Trusty kernel driver
   - Configure secure world memory

3. **Gatekeeper TA deployment**:
   - Build Gatekeeper Trusted Application
   - Sign TA with device-specific keys
   - Deploy to secure storage

4. **HAL replacement**:
   - Replace software gatekeeper HAL with Trusty version
   - Configure SELinux policies
   - Update device configuration

5. **Testing and validation**:
   - Verify CTS (Compatibility Test Suite) compliance
   - Run VTS (Vendor Test Suite) for HAL
   - Security testing and penetration testing

**Note**: Migration may require data migration strategy for existing users.

---

## 10. When to Use Which

### Choose AIDL Software Gatekeeper When:

- Developing/testing on Android Emulator
- Building proof-of-concept or prototype
- Hardware doesn't support TEE
- Security is not a primary concern
- Quick development iteration is needed
- Learning Android HAL architecture

### Choose Trusty TEE Gatekeeper When:

- Building production devices
- Security is critical
- Device handles sensitive user data
- Android CDD compliance required
- Device requires certification (Common Criteria, FIPS)
- Enterprise or payment use cases
- StrongBox Keymaster support needed

---

## 11. Related Android Security Components

Both gatekeeper implementations interact with:

- **Keymaster/Keystore**: Provides authentication tokens for key access
- **Lock Screen**: Uses gatekeeper for password verification
- **BiometricPrompt**: Integrates with gatekeeper for authentication
- **Synthetic Password**: Derives encryption keys using gatekeeper tokens

The security model choice (software vs. Trusty) affects the entire Android security stack.

---

## 12. Conclusion

The choice between **AIDL Software Gatekeeper** and **Trusty TEE Gatekeeper** represents a fundamental trade-off between simplicity and security:

- **Software Gatekeeper**: Fast, simple, portable – perfect for development but not secure enough for production
- **Trusty TEE Gatekeeper**: Secure, hardware-backed, compliant – more complex but essential for real-world devices

For Intel IoT devices in the vertical market, the choice depends on:
- **Use case**: Consumer device (Trusty) vs. development board (Software)
- **Threat model**: High-security environment (Trusty) vs. controlled environment (Software)
- **Hardware capabilities**: TEE available (Trusty) vs. no TEE (Software)
- **Compliance requirements**: Android CDD required (Trusty) vs. no certification needed (Software)

Most production Intel Android devices should use Trusty TEE Gatekeeper for proper security, while software gatekeeper remains valuable for development, testing, and specific low-security IoT applications.

---

## References

- [Android Hardware Gatekeeper HAL](https://source.android.com/docs/security/features/authentication/gatekeeper)
- [Trusty TEE](https://source.android.com/docs/security/features/trusty)
- [Android Compatibility Definition Document (CDD)](https://source.android.com/docs/compatibility/cdd)
- [ARM TrustZone Technology](https://developer.arm.com/ip-products/security-ip/trustzone)
- AIDL Software Implementation: `platform/hardware/interfaces/gatekeeper/aidl/software`
- Trusty Core: `platform/system/core/trusty`
