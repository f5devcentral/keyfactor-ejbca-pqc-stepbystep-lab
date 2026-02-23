# Module 02: EJBCA Configuration Management

## Understanding EJBCA's Configuration System

EJBCA keeps its configuration in a set of `.properties` files inside the `conf/` directory of the EJBCA source tree. Out of the box, these files are named `*.properties.sample` — you copy and rename them to `*.properties` to activate them.

This module explains each configuration file and what we'll set. We won't actually edit these files yet — that happens in Module 05 after we clone the EJBCA source code. But understanding what these files do *before* you start editing them makes the whole process much smoother.

> **📋 Keyfactor Reference:** [Manage EJBCA Configurations](https://docs.keyfactor.com/ejbca-software/latest/manage-ejbca-configurations)

---

## The Configuration Files

EJBCA uses five primary configuration files. Here's what each one does and what we'll configure:

### 1. install.properties — Management CA Settings

This file controls the initial Management CA that EJBCA creates during installation. The Management CA is an internal CA used to sign administrative certificates (TLS for the web UI, super admin client certificates, etc.).

**Key properties we'll set:**

| Property | Our Value | Purpose |
|----------|-----------|---------|
| `ca.name` | `ManagementCA` | Name of the internal management CA |
| `ca.dn` | `CN=ManagementCA,O=SassyCorp,C=US` | Subject DN for the management CA |
| `ca.tokentype` | `soft` | Software-based key storage (PKCS#12 in the database) |
| `ca.tokenpassword` | `null` | No password for soft key stores |
| `ca.keyspec` | `3072` | RSA 3072-bit keys for the Management CA |
| `ca.keytype` | `RSA` | RSA key type |
| `ca.signaturealgorithm` | `SHA256WithRSA` | Signature algorithm for the Management CA |
| `ca.validity` | `3650` | 10-year validity |
| `ca.certificateprofile` | `ROOTCA` | Use the built-in Root CA profile |

> **💡 Why RSA for the Management CA?** Great question. The Management CA is EJBCA's internal administrative CA — it handles TLS certificates for the web UI and admin authentication. We use RSA here because the web browsers and Java TLS stack connecting to EJBCA's admin interface need to trust these certificates, and browser support for ML-DSA in TLS is still emerging. Our *actual* PQC CAs (the SassyCorp Root and Intermediate) will be created natively in EJBCA in Module 07 using ML-DSA signing keys.

### 2. cesecore.properties — Core Security Settings

CESeCore is EJBCA's security core library. This file contains critical security configuration:

| Property | Our Value | Purpose |
|----------|-----------|---------|
| `password.encryption.key` | (random string) | Encrypts stored passwords in the database |
| `ca.rngalgorithm` | `SHA1PRNG` | Random number generation algorithm (FIPS compliant) |
| `ca.serialnumberoctetsize` | `20` | Size of certificate serial numbers (maximum per RFC 5280) |

> **⚠️ Important:** The `password.encryption.key` should be set before initial installation and **never changed afterward**. Changing it after installation would make existing encrypted passwords unreadable. Use a strong random string.

### 3. ejbca.properties — Application Settings

Controls EJBCA's application behavior:

| Property | Our Value | Purpose |
|----------|-----------|---------|
| `appserver.home` | `/opt/wildfly` | Points to the WildFly installation |
| `ejbca.productionmode` | `true` | Production mode (omits testing classes) |

### 4. web.properties — Web Interface & Admin Settings

Configures the EJBCA web interface, TLS, and the initial super administrator:

| Property | Our Value | Purpose |
|----------|-----------|---------|
| `java.trustpassword` | `changeit` | Password for the Java trust store |
| `superadmin.cn` | `SuperAdmin` | Common Name for the initial admin user |
| `superadmin.dn` | `CN=SuperAdmin,O=SassyCorp,C=US` | Full DN for the initial admin |
| `superadmin.password` | `ejbca` | Password for the admin PKCS#12 keystore |
| `superadmin.batch` | `true` | Auto-generate the admin keystore during install |
| `httpsserver.password` | `serverpwd` | Password for the TLS keystore |
| `httpsserver.tokentype` | `P12` | Keystore format — PKCS#12 (must match WildFly Elytron config) |
| `httpsserver.hostname` | `localhost` | Hostname for TLS certificate |
| `httpsserver.dn` | `CN=localhost,O=SassyCorp,C=US` | Subject DN for the TLS certificate |
| `httpserver.pubhttp` | `8080` | Public HTTP port |
| `httpserver.pubhttps` | `8442` | Public HTTPS port |
| `httpserver.privhttps` | `8443` | Private HTTPS port (admin access) |

> **💡 Port Breakdown:**
> - **8080** — Unencrypted HTTP (public enrollment pages, health checks)
> - **8442** — HTTPS without client cert requirement (public HTTPS access)
> - **8443** — HTTPS with client certificate authentication (admin UI — this is where you manage everything)

### 5. database.properties — Database Connection

Tells EJBCA how to connect to the database:

| Property | Our Value | Purpose |
|----------|-----------|---------|
| `datasource.jndi-name` | `EjbcaDS` | JNDI name matching the WildFly datasource |
| `database.name` | `mysql` | Database type (use `mysql` for MariaDB too) |

> **💡 Why `mysql` for MariaDB?** MariaDB is a drop-in replacement for MySQL and uses the same protocol. EJBCA's `database.name=mysql` works perfectly with MariaDB. The JDBC driver handles the rest.

---

## Configuration Strategy

Here's how the configuration workflow fits into our lab:

1. **Module 03** — We install MariaDB and create the database
2. **Module 04** — We install WildFly and configure the datasource
3. **Module 05** — We clone EJBCA, copy `.properties.sample` → `.properties`, edit each file, then build and deploy
4. **Module 06** — We run `ant runinstall` which reads the configuration and creates the Management CA

The actual file edits happen in Module 05. This module is your reference guide so you understand what each property does when you get there.

---

## Configuration File Location Map

After cloning EJBCA (Module 05), the configuration files will be at:

```
/opt/ejbca/conf/
├── install.properties.sample      → copy to install.properties
├── cesecore.properties.sample     → copy to cesecore.properties
├── ejbca.properties.sample        → copy to ejbca.properties
├── web.properties.sample          → copy to web.properties
├── database.properties.sample     → copy to database.properties
├── databaseprotection.properties.sample  (optional, skip for lab)
├── catoken.properties             (only needed for HSM, skip for lab)
└── ... (other optional configs)
```

---

## Security Considerations for Lab vs. Production

This is a learning lab, so we're using simple passwords and default settings in many places. In production, you'd want to:

| Lab Setting | Production Recommendation |
|-------------|--------------------------|
| `superadmin.password=ejbca` | Strong random password, stored in a vault |
| `httpsserver.password=serverpwd` | Strong random password |
| `java.trustpassword=changeit` | Strong random password |
| `ca.tokentype=soft` | HSM (PKCS#11) for all CA key storage |
| `httpsserver.hostname=localhost` | Actual FQDN with proper DNS |
| Passwords in plaintext configs | Use Elytron credential store (configured in Module 04) |

> **⚠️ Remember:** This lab is for educational purposes. The passwords and configurations here are intentionally simple so you can focus on learning the platform. Never use these values in a production PKI deployment please.

<br>

## What's Next

Now that you understand what EJBCA needs configured, let's start building the infrastructure. Next up: installing and configuring the database.

**Next Module:** [03 - Database Setup](03_ejbca_database.md)

---

*[← Previous: Module 01 - Prerequisites](01_ejbca_prerequisites.md) | [Next: Module 03 - Database →](03_ejbca_database.md)*
