# EJBCA Post-Quantum Cryptography Lab

Build quantum-resistant Certificate Authorities using **EJBCA Community Edition v9.3** and **NIST FIPS 204 ML-DSA** algorithms. By hand. Every command. No scripts, no Docker Compose, no "just run this one-liner." WE'RE going to learn this stuff whether we like it or not.

This is our second journy of the [Post-Quantum Cryptography Step-by-Step Lab](https://github.com/f5devcentral/openssl-pqc-stepbystep-lab) series. In Part 1 we built a PQC CA hierarchy from scratch using OpenSSL and learned what every flag, every file, and every config directive actually does. Here, you take that knowledge and build the same CAs inside an enterprise PKI platform — because knowing `openssl ca` flags by heart is great at parties but doesn't scale.

## What You'll Build

An entire PKI stack from the ground up:

- **MariaDB** database backend with proper character sets, binary log format, and both socket and TCP connectivity (because WildFly uses TCP and this will bite you exactly once before you learn why)
- **WildFly 35.0.1.Final** application server with 3-port TLS separation, Elytron security configuration, credential stores, and a JDBC datasource — all configured through JBoss CLI commands
- **EJBCA Community Edition v9.3** compiled from source with Apache Ant, deployed as an EAR, initialized with a Management CA, and serving a mutual TLS admin interface
- **ML-DSA-87 Root CA** created natively inside EJBCA's crypto token framework — quantum-resistant, NIST FIPS 204 compliant, same SassyCorp identity from the OpenSSL lab
- **ML-DSA-65 Intermediate CA** signed by the Root CA, completing a post-quantum chain of trust that you can verify with OpenSSL on the command line

When you're done, you'll have a web-based admin UI, certificate lifecycle management, audit logging, OCSP, CRL distribution, and a REST API — all backed by post-quantum cryptography. Disco.

## Why This Matters (The Short Version)

RSA and ECC have an expiration date. We don't know exactly when, but NIST has already finalized the replacements and everyone from the NSA, CCCS, to the BSI is publishing migration timelines. The transition is happening.

The problem is that reading about PQC migration and actually doing it are two completely different things. There's a gap between "I understand lattice-based cryptography conceptually" and "I can stand up a CA that signs certificates with ML-DSA-87 and I know which Bouncy Castle jar is going to ruin my afternoon." This lab closes that gap.

By the end, you'll have hands-on experience with the exact workflow organizations will need to follow: create crypto tokens with PQC key pairs, initialize certificate authorities with quantum-resistant signing algorithms, build a chain of trust, and manage it all through an enterprise platform. When someone in your organization asks "can we actually do this?" you'll be the one who already has.

## What is ML-DSA?

ML-DSA (Module-Lattice-Based Digital Signature Algorithm) is the NIST-standardized post-quantum signature scheme from [FIPS 204](https://csrc.nist.gov/pubs/fips/204/final). It replaces RSA and ECDSA for digital signatures in a post-quantum world. It comes in three flavors:

| Parameter Set | Security Level | What We Use It For |
|---------------|---------------|---------------------|
| **ML-DSA-44** | NIST Level 2 | Not used in this lab |
| **ML-DSA-65** | NIST Level 3 | Intermediate CA signing key |
| **ML-DSA-87** | NIST Level 5 | Root CA signing key |

ML-DSA-87 for the root (maximum security for the trust anchor), ML-DSA-65 for the intermediate (strong security, better performance for day-to-day signing). Same choices we made in the OpenSSL lab, now running inside PKI platform infrastructure.

## Things You'll Learn the Hard Way

This lab was tested, broken, fixed, retested, broken differently, and fixed again through many iterations. Here are some things we discovered so you don't have to:

- **EJBCA's `importca` can't handle ML-DSA keys yet.** The signer code path assumes RSA. We tried. It threw `BCMLDSAPrivateKey is not a RSAPrivateKey instance`. So we create natively instead. Every Keyfactor PQC example does it this way too.
- **WildFly ships its own Bouncy Castle** inside RESTEasy-Crypto. If you don't remove it, WildFly loads the old Bouncy Castle instead of EJBCA's PQC-capable version and you get cryptographic errors that are deeply unhelpful. Keyfactor notes this in their install.
- **`ant deploy-keystore` creates PKCS#12 files, not JKS.** If your Elytron key-store config says `type=JKS` or points to `.jks` files, nothing works. Ask me how many times we got this wrong.
- **WildFly's `launch.sh` uses `#!/bin/sh` but contains bash syntax.** On Ubuntu, `/bin/sh` is dash. The service starts, does nothing, exits silently. No error, no log. Just vibes.
- **The 3-port TLS setup is a manual step.** `ant deploy-keystore` creates keystores but does NOT configure Undertow listeners or Elytron SSL contexts. Without Module 04's configuration, port 8442 doesn't exist and port 8443 doesn't require client certificates. Your browser will never prompt you and you'll stare at a blank page wondering what you did wrong.
- **`/etc/sysconfig/` doesn't exist on Ubuntu** by default. WildFly's systemd service expects its config file there. If the directory is missing, WildFly starts with no variables set and immediately exits. Again, silently.

Every one of these is documented in the modules with explanations and fixes. You're welcome.

## Creating PQC Certificates for Your Environment

Once the lab is built, you have a fully operational PQC certificate authority. The Intermediate CA can sign end-entity certificates through EJBCA's enrollment interface, REST API, or CLI. Want to issue a quantum secure server certificate? Create a certificate profile, set the signature algorithm, and enroll. Want to test your application's PQC certificate handling? Issue certificates and export them. The infrastructure is there — the hard part is building it.

EJBCA also supports hybrid certificates that combine PQC and traditional algorithms, which is useful for environments that need to maintain backward compatibility during migration. The platform handles certificate lifecycle management, so you get renewal, revocation, CRL generation, and OCSP without managing flat files and serial numbers by hand like we did in the OpenSSL lab.

## Lab Modules

| Module | Description |
|--------|-------------|
| [00 - Introduction](/ejbca-pqc/00_ejbca_migration_intro.md.md) | Architecture overview, objectives, and why native creation |
| [01 - Prerequisites](/ejbca-pqc/01_ejbca_prerequisites.md.md) | OpenJDK 21, Apache Ant, system dependencies |
| [02 - Configurations](/ejbca-pqc/02_ejbca_configurations.md.md) | EJBCA configuration files and properties |
| [03 - Database](/ejbca-pqc/03_ejbca_database.md.md) | MariaDB setup, database creation, user privileges |
| [04 - WildFly](/ejbca-pqc/04_ejbca_wildfly.md.md) | WildFly 35 setup, datasource, 3-port TLS configuration |
| [05 - Deploy](/ejbca-pqc/05_ejbca_deploy.md.md) | Clone EJBCA CE v9.3, build with Ant, deploy |
| [06 - Install](/ejbca-pqc/06_ejbca_install.md.md) | Management CA, keystores, TLS initialization |
| [07 - Create PQC CAs](/ejbca-pqc/07_ejbca_create_pqc_ca.md.md) | Native ML-DSA Root and Intermediate CA creation |
| [08 - Finalize](/ejbca-pqc/08_ejbca_finalize.md.md) | Browser setup, verification, and testing |
| [09 - Reference](/ejbca-pqc/09_ejbca_reference.md.md) | Deployment reference, CLI cheat sheets, and troubleshooting |

## Prerequisites

You should seriously complete [Part 1 — the OpenSSL PQC Lab](https://github.com/f5devcentral/openssl-pqc-stepbystep-lab/tree/main/fipsqs) first. This lab builds on that knowledge and references the CA infrastructure you created there. If you skipped it, go back. We'll wait.

## Get Started

**[Start with Module 00 -->](/ejbca-pqc/00_ejbca_migration_intro.md)**

## References

| Resource | URL |
|----------|-----|
| NIST FIPS 204 (ML-DSA) | https://csrc.nist.gov/pubs/fips/204/final |
| EJBCA Community Edition | https://github.com/Keyfactor/ejbca-ce |
| EJBCA Installation Docs | https://docs.keyfactor.com/ejbca-software/latest/installation |
| Keyfactor PQC CA Tutorial | https://docs.keyfactor.com/ejbca/9.3.2/tutorial-create-pqc-hybrid-ca-chain |
| OpenSSL PQC Lab (Part 1) | https://github.com/f5devcentral/openssl-pqc-stepbystep-lab/tree/main/fipsqs |

---

*This lab is part of the [Post-Quantum Cryptography Step-by-Step Lab](https://github.com/f5devcentral/openssl-pqc-stepbystep-lab) series. For educational and internal testing purposes. Production deployments should use HSMs, air-gapped Root CAs, and follow organizational security policies.*