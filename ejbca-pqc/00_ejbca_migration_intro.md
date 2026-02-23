# Module 00: Building Enterprise PQC Certificate Authorities with EJBCA Community Edition

## Introduction — From OpenSSL Knowledge to Enterprise PKI Management

> **🎯 tl;dr - Our Goal:** Build enterprise-managed PQC Certificate Authorities in EJBCA, informed by everything we learned in the OpenSSL lab — same algorithms, same organizational identity, now with a PKI platform underneath.

Let's do more post-quantum cryptography! In our [NIST FIPS PQC Lab](https://github.com/f5devcentral/openssl-pqc-stepbystep-lab/tree/main/fipsqs), we built a fully functional quantum-resistant Certificate Authority using OpenSSL 3.5.x — complete with an ML-DSA-87 Root CA, an ML-DSA-65 Intermediate CA, end-entity certificates, and revocation infrastructure. Fun stuff.

Now comes a real-world question: **how do you manage that CA infrastructure at scale?**

OpenSSL is an incredible tool for building and understanding PKI, but operating a production CA requires lifecycle management, role-based access control, audit logging, OCSP responders, CRL distribution, and management that doesn't involve memorizing `openssl ca` flags. That's where PKI management solutions like [EJBCA Community Edition](https://www.ejbca.org/) come in.

This lab guide walks you through building **new, enterprise-grade PQC Certificate Authorities** natively inside **Keyfactor's EJBCA Community Edition v9.3** — an open-source, PKI management platform. Apply knowledge from the prior learnings in the OpenSSL lab (algorithm choices, key sizes, subject DNs, chain-of-trust design) to create a certificate authority with the same SassyCorp identity, now backed by a proper PQC PKI solution. 

By the end, you'll have quantum-resistant Root and Intermediate CAs managed through EJBCA with certificate lifecycle management, audit logging, and a web-based admin interface — and you'll be more aligned to where we need to end up in order to start practicing cryptographic agility; crypto agility for the tech term marketing people. It's going to be a hotter topic as we ebb and flow with the quantum resistant crypto-secure future being mandated by pretty much everyone at this point. So suck up the marketing terms and let's dive in.

<br>

## Why Native Creation Instead of Import?

You might be wondering: "We already built PQC CAs in OpenSSL — why not just import those keys and certificates into EJBCA?"

Great question. We tried. Here's what happens:

EJBCA's `importca` command packages external CA keys and certificates into PKCS#12 keystores and imports them as soft crypto tokens. The problem is that the code path that creates the signer during import **assumes RSA keys**. When you hand it an ML-DSA key, it throws:

```
OperatorCreationException: cannot create signer: Supplied key (BCMLDSAPrivateKey) is not a RSAPrivateKey instance
```

This is an current limitation in EJBCA Community Edition — the `importca` signer code path casts to `RSAPrivateKey` internally. It's not a bug in the PQC algorithm support itself; EJBCA's Bouncy Castle provider handles ML-DSA just fine for **native** crypto token operations. The import path just hasn't been updated for non-RSA key types yet.

All of Keyfactor's own PQC examples (including their [PQC Hybrid CA Tutorial](https://docs.keyfactor.com/ejbca/9.0/tutorial-create-pqc-hybrid-ca-chain)) use **native crypto token creation** inside EJBCA, never external import. So that's what we'll do too.

The good news: everything you learned in the OpenSSL lab directly informs how we configure these CAs. Same algorithm choices (ML-DSA-87 for Root, ML-DSA-65 for Intermediate), same SassyCorp organizational identity, same chain-of-trust architecture. We're just building them inside EJBCA's crypto token framework instead of importing them from outside.

> **💡 Your OpenSSL CAs aren't wasted.** They taught you how PQC certificate authorities work at the deepest level — key generation, certificate signing, chain validation, revocation. That knowledge is exactly what makes this lab possible. You'll know *why* every configuration choice matters because you've already done it by hand.

<br>

## What is EJBCA?

EJBCA (Enterprise Java Beans Certificate Authority) is an open-source PKI platform developed by Keyfactor. It's one of the more widely deployed PKI platforms and solves for:

- **Certificate Lifecycle Management** — issuance, renewal, revocation, and expiration tracking
- **Protocol Support** — SCEP, CMP, EST, ACME, OCSP, and REST APIs
- **Role-Based Access Control** — granular permissions for administrators and operators
- **Audit Logging** — comprehensive, tamper-evident logging of all PKI operations
- **Crypto Token Management** — soft tokens, PKCS#11, and HSM integration

The Community Edition (CE) provides the core CA functionality we need for this lab. The Enterprise Edition adds features like HA, cloud HSM support, and advanced compliance tools — but CE is more than sufficient for learning and many environments.

<br>

## What We're Building

This lab follows the official [Keyfactor EJBCA installation documentation](https://docs.keyfactor.com/ejbca-software/latest/installation) and extends it with native PQC CA creation. Here's the full stack:

```
┌──────────────────────────────────────────────┐
│       EJBCA Admin Web UI (Browser)           │
│  Port 8080 — HTTP (public pages)             │
│  Port 8442 — HTTPS (server auth only)        │
│  Port 8443 — HTTPS (mTLS, client cert REQ)   │
├──────────────────────────────────────────────┤
│        EJBCA Community Edition v9.3          │
│     (git clone from Keyfactor/ejbca-ce)      │
├──────────────────────────────────────────────┤
│         WildFly 35.0.1.Final App Server      │
│         (Jakarta EE Platform)                │
├──────────────────────────────────────────────┤
│      OpenJDK 21 (ARM64 or AMD64)             │
├──────────────────────────────────────────────┤
│         MariaDB Database                     │
└──────────────────────────────────────────────┘
```

### The 3-Port Architecture

EJBCA uses three distinct ports, each with a different security model. Understanding this upfront saves a lot of confusion later:

| Port | Protocol | TLS Mode | Purpose |
|------|----------|----------|---------|
| **8080** | HTTP | None | Public enrollment pages, health checks, unencrypted access |
| **8442** | HTTPS | Server auth only | Public HTTPS access — server presents its certificate, no client cert required |
| **8443** | HTTPS | Mutual TLS (mTLS) | Admin web interface — server presents its cert AND requires a client certificate from your browser |

Port 8443 is where all the admin magic happens. When you navigate to `https://localhost:8443/ejbca/adminweb/`, WildFly demands your browser present a client certificate (the SuperAdmin cert we'll generate). If your browser doesn't have one, you get rejected. This is mutual TLS authentication — both sides prove their identity.

> **⚠️ Important:** The `ant deploy-keystore` command (which we run during installation) creates the TLS keystores and a basic SSL context called `applicationSSC` — but it does **NOT** configure the 3-port Undertow/Elytron listener setup. That's a separate manual step we handle in Module 04. Without it, port 8442 won't exist and port 8443 won't require client certificates, which means Firefox will never prompt you for a certificate and you'll never reach the admin UI.

<br>

## Prerequisites

### Completed Prior Lab

You **must** have completed the [NIST FIPS PQC Lab](https://github.com/f5devcentral/openssl-pqc-stepbystep-lab/tree/main/fipsqs) before starting this guide. Specifically, you should have:

- 🌈 An ML-DSA-87 Root CA with key and certificate at `/opt/sassycorp-pqc/root-ca/`
- 🌈 An ML-DSA-65 Intermediate CA with key and certificate at `/opt/sassycorp-pqc/intermediate-ca/`
- 🌈 A working certificate chain (Root → Intermediate → End Entity)
- 🌈 OpenSSL 3.5.x with native PQC algorithm support

If you don't have these, go back and complete the FIPS lab first. We'll wait. I'll grab some coffee. ☕ 

*Note: I'm tired of AI ruining green checks so I'm using rainbows now. Bring it.*

### System Requirements

| Component | Requirement |
|-----------|-------------|
| **Operating System** | Ubuntu 25.10 (same VM/system from prior lab) |
| **Architecture** | AMD64 (x86_64) or ARM64 (aarch64) — both supported |
| **RAM** | 4 GB minimum (8 GB recommended — WildFly alone wants 2 GB heap) |
| **Permissions** | Root or sudo access |
| **Network** | Internet access for package downloads, git clone, and for taking breaks on slashdot |

### Software Stack

| Software | Version | Purpose |
|----------|---------|---------|
| **OpenJDK** | 21 (21.0.5+) | Java runtime for EJBCA and WildFly |
| **WildFly** | 35.0.1.Final | Jakarta EE application server |
| **MariaDB** | Latest from Ubuntu repos | Database backend for EJBCA |
| **Apache Ant** | 1.9.8+ | Build tool for EJBCA compilation and deployment |
| **Git** | Latest from Ubuntu repos | Clone EJBCA CE source code |
| **EJBCA CE** | 9.3 | The PKI platform itself |

### Required Knowledge

- Everything from the prior lab (Linux CLI, PKI concepts, X.509 structure)
- Basic understanding of Java application servers (helpful but not required — we'll explain as we go)
- Familiarity with database concepts (tables, users, privileges)

<br>

## Architecture Support — ARM64 and AMD64

This lab supports both ARM64 (aarch64) and AMD64 (x86_64) architectures. OpenJDK 21 provides native builds for both. Throughout this guide, when a step differs between architectures, we'll call it out explicitly. In most cases, the installation is identical.

To check your architecture:

```bash
uname -m
```

**Expected Output:**
- `x86_64` — you're on AMD64
- `aarch64` — you're on ARM64

### Where Our OpenSSL PQC CAs Currently Resides (Reference)

From the prior lab, your SassyCorp CA infrastructure should still be at:

```
/opt/sassycorp-pqc/
├── root-ca/
│   ├── certs/
│   │   └── root-ca.crt          ← ML-DSA-87 Root CA certificate
│   ├── private/
│   │   └── root-ca.key          ← ML-DSA-87 Root CA private key
│   ├── crl/
│   ├── newcerts/
│   ├── index.txt
│   └── serial
└── intermediate-ca/
    ├── certs/
    │   ├── intermediate-ca.crt  ← ML-DSA-65 Intermediate CA certificate
    │   └── ca-chain.crt         ← Full certificate chain
    ├── private/
    │   └── intermediate-ca.key  ← ML-DSA-65 Intermediate CA private key
    ├── crl/
    ├── newcerts/
    ├── index.txt
    └── serial
```

> **💡 Note:** We won't be importing these files into EJBCA (see "Why Native Creation" above), but they serve as our reference for the organizational identity and algorithm choices we'll replicate. Your OpenSSL CAs remain intact as a working reference and backup/secondary CA.

<br>

## Manual Entry — Not Scripts

Just like the OpenSSL PQC lab, this guide uses **manual command entry only — no scripts**. No Docker Compose, no signing up for prebuilt configurations. Every command you type, every configuration file you edit, every CLI interaction is done by hand. Blame our love for "Learning Python the Hard Way" for this method.

Why? Because:

1. **You learn the tools.** You'll understand WildFly's JBoss CLI, EJBCA's Ant commands, MariaDB's SQL interface, and systemd service management.
2. **You learn the "why."** Every command has context explaining what it does and why it matters.
3. **You can troubleshoot.** When something goes wrong (and in PKI, something always goes wrong), you'll know where to look because you built it yourself.
4. **You remember it.** Muscle memory from manual entry beats copy-paste amnesia every time.

You COULD copy/paste from this guide but you're only cheating yourself.

<br>

## What's Different from Keyfactor's Official Docs?

Keyfactor's official documentation is plenty great, they have a lot of options — it's our reference throughout this lab. But we've adapted it for our specific use case:

| Official Docs | Our Lab |
|--------------|---------|
| Generic CA setup | PQC CAs created natively using OpenSSL lab knowledge |
| Multiple database options | MariaDB only (focused and clear) |
| Multiple app servers | WildFly 35 only |
| Production-oriented | Learning-oriented with explanations |
| RSA/ECDSA examples | Post-quantum (ML-DSA) throughout |

We follow the same order and structure as the official docs, but every step includes the educational context and `SassyCorp`-specific configuration that makes this a learning experience rather than a deployment checklist.

<br>

## A Note on Post-Quantum Compatibility

Here's something important: EJBCA uses Bouncy Castle as its cryptographic provider, and Bouncy Castle has been adding PQC algorithm support. EJBCA CE v9.3 includes Bouncy Castle libraries that support ML-DSA and other post-quantum algorithms.

When creating PQC CAs natively in EJBCA, there are a few things to be aware of:

- **Bouncy Castle version conflicts** — WildFly ships its own Bouncy Castle, which can conflict with EJBCA's bundled version (we solve for this in Module 04)
- **Signing vs. encryption key pairs** — PQC signing keys (ML-DSA) work great, but EJBCA requires the encryption key pair to still be RSA (per Keyfactor's documentation)
- **OCSP signing** — doesn't support PQC yet; must use RSA/EC for OCSP signing keys

We'll address each of these in detail as we encounter them. For now, just be aware that PQC + EJBCA is bleeding edge territory and we'll be working through any compatibility considerations together. As the software revs, we'll keep updating this guide.

<br>

## Enough already, let's Go

Ready to build enterprise-grade, quantum-resistant Certificate Authorities?

**Next Module:** [01 - Installation Prerequisites](01_ejbca_prerequisites.md)

<br>

## References

| Resource | URL |
|----------|-----|
| EJBCA Community Edition Source | https://github.com/Keyfactor/ejbca-ce |
| EJBCA Software Stack Documentation | https://docs.keyfactor.com/ejbca-software/latest/installation |
| OpenSSL PQC Lab (Prior Lab) | https://github.com/f5devcentral/openssl-pqc-stepbystep-lab/tree/main/fipsqs |
| WildFly 35 | https://www.wildfly.org/ |
| OpenJDK 21 | https://openjdk.org/projects/jdk/21/ |
| MariaDB | https://mariadb.org/ |
| NIST FIPS 204 (ML-DSA) | https://csrc.nist.gov/pubs/fips/204/final |
| Keyfactor EJBCA PQC Hybrid Tutorial | https://docs.keyfactor.com/ejbca/9.0/tutorial-create-pqc-hybrid-ca-chain |

---

*This lab is part of the [Post-Quantum Cryptography Step-by-Step Lab](https://github.com/f5devcentral/openssl-pqc-stepbystep-lab) series. For educational and internal testing purposes. Production deployments should use HSMs, air-gapped Root CAs, and follow organizational security policies.*
