# Module 01: Installation Prerequisites

Before EJBCA can run, it needs a foundation. EJBCA is a Java application that runs inside an application server (WildFly), stores its data in database (MariaDB), and is compiled from source using Apache Ant. Each of these has its own dependencies.

This module installs everything EJBCA needs to compile, deploy, and run — except for EJBCA itself, the database, and the application server, which each get their own modules. We provide all the instructions but you can check it out on Keyfactor's web site if you want.

> **📋 Keyfactor Reference:** [Installation Prerequisites](https://docs.keyfactor.com/ejbca-software/latest/installation-prerequisites)

<br>

## Step 1: Update Your System

Start by making sure your Ubuntu system is current. This is the same system you used for the OpenSSL PQC lab, so it should already be in good shape. Mostly.

```bash
sudo apt update && sudo apt upgrade -y
```

<br>

## Step 2: Install System Dependencies

EJBCA and its supporting tools need several system packages, most of these we have but we're being safe in case you're using your own linux flavor:

```bash
sudo apt install -y \
    curl \
    wget \
    git \
    ca-certificates
```

**What each package provides:**

| Package | Purpose |
|---------|---------|
| `curl`, `wget` | Downloading files (WildFly, JDBC drivers, etc.) |
| `git` | Cloning the EJBCA CE source code from GitHub |
| `ca-certificates` | System root CA certificates for HTTPS connections |

<br>

## Step 3: Install OpenJDK 21

EJBCA requires Java 21 (specifically version 21.0.5 or later). OpenJDK 21 is the recommended JDK per Keyfactor's documentation (I did try openjdk-26-jdk and it seemed fine but let's not start playing variable scrabble).

### Install from Ubuntu Repositories

```bash
sudo apt install -y openjdk-21-jdk
```

### Verify the Installation

```bash
java -version
```

**Expected Output (will vary slightly by build):**
```
openjdk version "21.0.x" 202x-xx-xx
OpenJDK Runtime Environment (build 21.0.x+xx-Ubuntu-...)
OpenJDK 64-Bit Server VM (build 21.0.x+xx-Ubuntu-..., mixed mode, sharing)
```

The critical part is that it shows version **21.0.5 or later**.

### Set JAVA_HOME

EJBCA and its build tools need the `JAVA_HOME` environment variable. Let's set it permanently:

```bash
echo 'export JAVA_HOME=/usr/lib/jvm/java-21-openjdk-'$(dpkg --print-architecture) | sudo tee /etc/profile.d/java.sh
```

Load it into your current session:

```bash
source /etc/profile.d/java.sh
```

Verify it's set:

```bash
echo $JAVA_HOME
```

**Expected Output (AMD64):**
```
/usr/lib/jvm/java-21-openjdk-amd64
```

**Expected Output (ARM64):**
```
/usr/lib/jvm/java-21-openjdk-arm64
```

> **💡 Architecture Note:** The `dpkg --print-architecture` command automatically detects whether you're on `amd64` or `arm64`, so the same command works on both architectures. OpenJDK 21 has native builds for both. I forget this so many times and when it bites halfway through an install I have no one to blame but Jason Rahm. 🔥

### Verify the Java Compiler

We need the JDK (not just JRE) because we're compiling EJBCA from source:

```bash
javac -version
```

**Expected Output:**
```
javac 21.0.x
```

---

## Step 4: Install Apache Ant

Apache Ant is the build tool EJBCA uses to compile its source code and deploy the application. Keyfactor requires Ant 1.9.8 or later.

```bash
sudo apt install -y ant
```

### Verify the Installation

```bash
ant -version
```

**Expected Output:**
```
Apache Ant(TM) version 1.10.x compiled on ...
```

Any version 1.9.8 or higher is fine.

> **💡 Why Ant?** You might wonder why EJBCA uses Ant instead of Maven or Gradle. EJBCA has been around for a bit and Ant was the standard Java build tool at the time. The build system is deeply integrated with EJBCA's deployment workflow — `ant deployear`, `ant runinstall`, `ant deploy-keystore` are all EJBCA-specific Ant targets that handle compilation, deployment, and PKI initialization. It works, and it works well.

<br>

## Step 5: Verify Your Existing PQC Infrastructure

Before we move forward, let's confirm your PQC CA from the prior lab is intact and accessible.

### Check the Root CA Certificate

```bash
openssl x509 -in /opt/sassycorp-pqc/root-ca/certs/root-ca.crt -noout -subject -issuer
```

**Expected Output:**
```
subject=C=US, ST=Washington, L=Glacier, O=SassyCorp, OU=IT Security Division, CN=SassyCorp Root CA, emailAddress=ca-admin@sassycorp.internal
issuer=C=US, ST=Washington, L=Glacier, O=SassyCorp, OU=IT Security Division, CN=SassyCorp Root CA, emailAddress=ca-admin@sassycorp.internal
```

### Verify ML-DSA-87 Signature

```bash
openssl x509 -in /opt/sassycorp-pqc/root-ca/certs/root-ca.crt -noout -text | grep "Signature Algorithm"
```

**Expected Output:**
```
    Signature Algorithm: ML-DSA-87
        Signature Algorithm: ML-DSA-87
```

### Check the Intermediate CA Certificate

```bash
openssl x509 -in /opt/sassycorp-pqc/intermediate-ca/certs/intermediate-ca.crt -noout -subject -issuer
```

### Verify ML-DSA-65 Signature

```bash
openssl x509 -in /opt/sassycorp-pqc/intermediate-ca/certs/intermediate-ca.crt -noout -text | grep "Signature Algorithm"
```

**Expected Output:**
```
    Signature Algorithm: ML-DSA-65
        Signature Algorithm: ML-DSA-87
```

Note: The first line is the Intermediate CA's own algorithm (ML-DSA-65). The second is the algorithm the Root CA used to sign it (ML-DSA-87). This is expected.

### Check Private Keys Exist

```bash
sudo ls -la /opt/sassycorp-pqc/root-ca/private/root-ca.key
sudo ls -la /opt/sassycorp-pqc/intermediate-ca/private/intermediate-ca.key
```

*Note: We sudo `ls` because those keys were protected under the pqcadmin user.*

Both files should exist. If they've been moved, locate them now.

### Verify the Certificate Chain

```bash
openssl verify -CAfile /opt/sassycorp-pqc/root-ca/certs/root-ca.crt /opt/sassycorp-pqc/intermediate-ca/certs/intermediate-ca.crt
```

**Expected Output:**
```
/opt/sassycorp-pqc/intermediate-ca/certs/intermediate-ca.crt: OK
```

If this says OK, your certificate chain is intact. Your OpenSSL PQC CAs are healthy and will serve as the reference for the CAs we build natively in EJBCA.

<br>

## Step 6: Create Directory Structure

Let's create the directories we'll use throughout this lab:

```bash
sudo mkdir -p /opt/ejbca
```

**Directory purposes:**

| Directory | Purpose |
|-----------|---------|
| `/opt/ejbca` | EJBCA source code and build artifacts |
| `/opt/wildfly` | WildFly application server (created as symlink in Module 04) |
| `/opt/sassycorp-pqc/` | Existing PQC CA infrastructure from prior lab (reference) |

---

## Checkpoint

Before moving to the next module, verify everything is in place:

```bash
echo "=== Java ===" && java -version 2>&1 | head -1
echo "=== Java Compiler ===" && javac -version
echo "=== JAVA_HOME ===" && echo $JAVA_HOME
echo "=== Ant ===" && ant -version 2>&1 | head -1
echo "=== Git ===" && git --version
echo "=== Architecture ===" && uname -m
```

All items should return valid output. If anything is missing, go back and fix it before proceeding.

<br>

## Troubleshooting

### PQC certificates not found

If your certificates aren't at the expected paths:

```bash
sudo find / -name "root-ca.crt" -type f 2>/dev/null
sudo find / -name "intermediate-ca.crt" -type f 2>/dev/null
```

---

**Next Module:** [02 - EJBCA Configuration Management](02_ejbca_configurations.md)

---

*[← Previous: Module 00 - Introduction](00_ejbca_migration_intro.md) | [Next: Module 02 - Configurations →](02_ejbca_configurations.md)*
