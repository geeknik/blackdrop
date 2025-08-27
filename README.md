# Blackdrop

> **Cypherpunk dead-drop for one-shot payloads. No accounts, no logs, no metadata (by design), zero server-held keys, and retrieve-once semantics.**

[![License: MIT](https://img.shields.io/badge/License-MIT-blue.svg)](LICENSE)
[![Rust](https://img.shields.io/badge/rust-2021-orange.svg)](https://www.rust-lang.org)
[![Security: Experimental](https://img.shields.io/badge/security-experimental-red.svg)](#security-warnings)

Blackdrop is an end-to-end encrypted dead-drop system for securely sharing ephemeral data between parties. It implements aggressive metadata minimization, retrieve-once semantics, and operates over Tor by default for maximum privacy.

## What is Blackdrop?

Blackdrop is a **minimal, asynchronous dead-drop application** that enables secure delivery of short notes or files with:

- **End-to-end encryption** - Keys never touch the server
- **Encryption at rest** - Server stores only padded ciphertext blobs  
- **Metadata minimization** - No accounts, logs, or correlatable information
- **Retrieve-once deletion** - Data is destroyed after first access
- **Network privacy** - Tor onion service by default
- **Verifiable erasure** - Cryptographic proof of deletion

The server acts as a **blind storage node** that cannot decrypt, index, or meaningfully fingerprint payloads. Each drop is a single-use envelope with configurable TTL and automatic shredding.

## Key Security Features

### Cryptographic Stack
- **Key Exchange**: X25519 (ECDH) with Noise NK protocol pattern
- **Signatures**: Ed25519 for authentication  
- **AEAD**: XChaCha20-Poly1305 (wide nonce, misuse resistant)
- **Hashing**: BLAKE3 for all hash operations
- **Key Derivation**: HKDF with BLAKE3
- **Post-Compromise Security**: Key ratcheting for multi-chunk files

### Privacy & Metadata Protection
- **Uniform padding** to fixed sizes (256 KiB, 1 MiB, 4 MiB) reduces fingerprinting
- **No server-side plaintext** - all encryption/decryption happens client-side
- **Sealed-sender support** - optional sender anonymity within encrypted payload
- **Traffic shaping** - randomized timing and cover traffic
- **Clock fuzzing** - coarse timestamp buckets prevent timing correlation

### Data Protection
- **Memory safety** - secure zeroization of sensitive data using `zeroize` crate
- **Constant-time operations** where possible to prevent timing attacks
- **Full-disk encryption required** for server storage (LUKS2/XTS-AES recommended)
- **Verifiable deletion** with cryptographic receipts

## Quick Start

### Prerequisites

- **Rust 1.75+** 
- **Tor daemon** 
- **Linux/macOS** (Fuck Windows)

### Building from Source

```bash
# Clone the repository
git clone https://github.com/blackdrop/blackdrop.git
cd blackdrop

# Build all components
cargo build --release

# Run tests
cargo test

# Install client binary
cargo install --path blackdrop-client
```

### Server Setup (Optional - for self-hosting)

```bash
# Build server
cargo build --release --bin blackdrop-server

# Set up encrypted storage volume
sudo cryptsetup luksFormat /dev/sdb1
sudo cryptsetup luksOpen /dev/sdb1 blackdrop-data

# Create configuration
sudo mkdir -p /etc/blackdrop
sudo tee /etc/blackdrop/config.toml << EOF
[server]
bind_addr = "127.0.0.1:8080"
data_dir = "/var/blackdrop/data"
max_payload_size = "32MiB"
default_ttl_hours = 24
max_ttl_hours = 168

[tor]
onion_service_dir = "/var/lib/tor/blackdrop"
hidden_service_port = 80
EOF

# Run server
./target/release/blackdrop-server
```

## CLI Usage Examples

### Creating a Drop

```bash
# Create drop from file
blackdrop put document.pdf
# Outputs: Capability token + QR code

# Create drop from stdin
echo "Secret message" | blackdrop put -
# Outputs: bd1_aBcDeFgHiJkLmNoPqRsTuVwXyZ1234567890_example

# Custom TTL (max 168 hours = 7 days)
blackdrop put --ttl 48 secret.txt

# Skip QR code display
blackdrop put --no-qr data.zip
```

### Retrieving a Drop

```bash
# Retrieve to stdout
blackdrop get bd1_aBcDeFgHiJkLmNoPqRsTuVwXyZ1234567890_example

# Save to file
blackdrop get bd1_aBcDeFgHiJkLmNoPqRsTuVwXyZ1234567890_example -o received.pdf

# Use custom server
blackdrop get --server http://custom.onion bd1_token
```

### Burning a Drop (Delete Before Retrieval)

```bash
# Delete drop permanently
blackdrop burn bd1_aBcDeFgHiJkLmNoPqRsTuVwXyZ1234567890_example
```

## Security Model

### Threat Model

**Protected Against:**
- Global passive observer (GPO) through Tor routing
- Malicious storage operator - server cannot decrypt data
- Network active attackers - authenticated encryption prevents tampering  
- Legal compulsion - server has no keys or plaintext to surrender
- Forensic analysis after TTL - verifiable secure deletion

**Assumptions:**
- Client devices are not compromised during operation
- Users can access Tor network reliably
- Server operator follows disk encryption best practices
- Binary obtained through trusted channel (reproducible builds)

**Out of Scope:**
- Endpoint compromise (keyloggers, malware)
- Physical device seizure before data deletion
- Side-channel attacks from compromised OS
- Global active network attacks without Tor

### Data Flow Security

1. **Client-side encryption** - All crypto operations happen locally
2. **Padded transport** - Uniform blob sizes prevent size correlation
3. **Blind server storage** - Server sees only encrypted, padded chunks
4. **Retrieve-once deletion** - First successful retrieval triggers shredding
5. **TTL enforcement** - Automatic expiry with cryptographic proof
6. **Memory hygiene** - Sensitive data zeroized after use

## Build Requirements

### System Dependencies

```bash
# Ubuntu/Debian
sudo apt update
sudo apt install build-essential pkg-config libssl-dev tor

# macOS (via Homebrew)
brew install rust tor

# Arch Linux
sudo pacman -S rust tor openssl pkgconf
```

### Rust Dependencies

The project uses a workspace structure with these main crates:
- `blackdrop-core` - Cryptographic primitives and envelope handling
- `blackdrop-client` - CLI tool with Tor integration
- `blackdrop-server` - Relay/storage node (optional for self-hosting)

Key external dependencies:
- `chacha20poly1305` - AEAD encryption
- `x25519-dalek` + `ed25519-dalek` - Elliptic curve cryptography
- `snow` - Noise protocol implementation
- `blake3` - Fast cryptographic hashing
- `arti-client` - Pure Rust Tor client library
- `axum` - Web framework for server component

## Security Warnings

### 🚨 MVP Status - Not Production Ready

**This is experimental software under active development:**

- ❌ **No security audit completed** - Cryptographic implementation not professionally reviewed
- ❌ **Memory safety not verified** - Side-channel resistance needs validation  
- ❌ **Protocol not formally verified** - Threat model assumptions require proof
- ❌ **Limited testing** - Insufficient coverage for production deployment
- ❌ **API may change** - Breaking changes expected before v1.0

### Current Limitations

- **SSD forensics**: Secure deletion relies on full-disk encryption + TRIM, not guaranteed
- **Timing attacks**: Padding/jitter reduces but doesn't eliminate all timing leaks
- **Denial of service**: PoW throttling helps but cannot prevent determined floods
- **UX brittleness**: Retrieve-once semantics are unforgiving if client fails

### Operational Security

If you choose to experiment with Blackdrop:

- ✅ **Use Tor** - Never operate over clearnet
- ✅ **Encrypt server storage** - LUKS2/FileVault2 minimum
- ✅ **Disable logging** - No application or system logs on server
- ✅ **Isolated environment** - Dedicated VM/container for server
- ✅ **Regular updates** - Monitor repository for security patches
- ✅ **Test thoroughly** - Verify deletion and TTL behavior
- ✅ **Have backups** - Retrieve-once means data loss if client fails

## Contributing

We welcome contributions but please note this is **experimental cryptographic software**. 

### Development Setup

```bash
git clone https://github.com/blackdrop/blackdrop.git
cd blackdrop

# Install development tools
cargo install cargo-audit cargo-watch

# Run continuous tests during development
cargo watch -x test

# Check for security advisories
cargo audit
```

### Security Review Process

- All cryptographic code requires review by multiple contributors
- Memory safety verified with Valgrind/AddressSanitizer where possible
- Constant-time verification for sensitive operations
- Fuzzing for envelope parsing and network protocol handling

## License

Licensed under either of:

- Apache License, Version 2.0 ([LICENSE-APACHE](LICENSE-APACHE) or http://www.apache.org/licenses/LICENSE-2.0)
- MIT License ([LICENSE-MIT](LICENSE-MIT) or http://opensource.org/licenses/MIT)

at your option.

## Disclaimer

This software is provided "as is" without warranty. The authors are not responsible for any damages resulting from use. Blackdrop is designed for legitimate privacy protection - users are responsible for compliance with local laws.

**Do not rely on Blackdrop to protect against compromised endpoints or negligent OPSEC.**

---

*For detailed design documentation, see [DESIGN.md](DESIGN.md)*
