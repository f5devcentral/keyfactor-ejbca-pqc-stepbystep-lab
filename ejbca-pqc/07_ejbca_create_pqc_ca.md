# Module 07: Create PQC Certificate Authorities in EJBCA

## The Main Event

This is why we're here. Everything we've built so far — the database, WildFly, EJBCA with its Management CA — was all scaffolding for this moment. We're going to create quantum-resistant Certificate Authorities **natively** inside EJBCA using the same design and identity from our OpenSSL lab.

After this module, you'll have ML-DSA-87 Root CA and ML-DSA-65 Intermediate CA managed entirely by EJBCA — with certificate lifecycle management, audit logging, and a web interface.

> **📋 Keyfactor Reference:** [Tutorial: Build a Post-Quantum Ready PKI](https://docs.keyfactor.com/ejbca/9.3.2/tutorial-build-a-post-quantum-ready-pki) — our approach follows Keyfactor's native crypto token creation pattern, adapted for CLI.

---

## Why Native Creation?

As we covered in Module 00, EJBCA's `importca` command can't handle ML-DSA keys — the signer code path assumes RSA (`BCMLDSAPrivateKey is not a RSAPrivateKey instance`). All of Keyfactor's PQC examples use native crypto token creation, and that's what works.

The good news: we're not starting from scratch intellectually. Every decision we make here — algorithm choices, subject DNs, key sizes, chain structure — comes directly from what we built and validated in the OpenSSL lab. We're just building them inside EJBCA's framework instead of importing from outside.

---

## A Note on CLI Commands

Keyfactor's PQC tutorials demonstrate CA creation through the Admin Web UI. We're using the EJBCA CLI (`ejbca.sh`) instead, consistent with our lab's manual command-entry philosophy. The CLI switch names can vary between EJBCA versions — if any command fails, run it with `--help` to verify the exact syntax for your version:

```bash
sudo bin/ejbca.sh ca init --help
sudo bin/ejbca.sh cryptotoken create --help
sudo bin/ejbca.sh cryptotoken generatekey --help
```

> **⚠️ Named switches:** We use named switches (`--caname`, `--dn`, etc.) for all CLI commands instead of positional arguments. Positional args are fragile and easy to get wrong — named switches are explicit and self-documenting. Lesson learned from testing.

---

## Step 1: Create the Root CA Crypto Token

A crypto token in EJBCA is a key container — it holds the cryptographic keys a CA uses for signing. We'll create a soft crypto token (software-based, keys stored in the database) with ML-DSA-87 as the signing algorithm.

```bash
cd /opt/ejbca
```

### Create the Crypto Token

```bash
sudo bin/ejbca.sh cryptotoken create \
    --token "SassyCorp Root CA Token" \
    --type "SoftCryptoToken" \
    --pin "foo123" \
    --autoactivate "true"
```

**Parameters explained:**

| Parameter | Value | Purpose |
|-----------|-------|---------|
| `--token` | `SassyCorp Root CA Token` | Display name for the crypto token |
| `--type` | `SoftCryptoToken` | Software-based key storage (PKCS#12 in database) |
| `--pin` | `foo123` | Activation PIN for the token (lab value) |
| `--autoactivate` | `true` | Token activates automatically on WildFly restart |

### Generate the ML-DSA-87 Signing Key Pair

```bash
sudo bin/ejbca.sh cryptotoken generatekey \
    --token "SassyCorp Root CA Token" \
    --alias "signKey" \
    --keyspec "ML-DSA-87"
```

> **💡 This is the PQC moment.** EJBCA's Bouncy Castle provider creates a native ML-DSA-87 key pair directly inside the crypto token. No external import needed.

### Generate the RSA Encryption Key Pair

Per Keyfactor's documentation, the encryption key pair must still be RSA — PQC encryption (ML-KEM) isn't supported in EJBCA's CA operations yet. This key is used for encrypting stored data, not for signing certificates.

```bash
sudo bin/ejbca.sh cryptotoken generatekey \
    --token "SassyCorp Root CA Token" \
    --alias "encryptKey" \
    --keyspec "4096"
```

### Verify the Token and Keys

```bash
sudo bin/ejbca.sh cryptotoken listkeys --token "SassyCorp Root CA Token"
```

**Expected Output:** You should see two keys listed: `signKey` (ML-DSA-87) and `encryptKey` (RSA 4096).

---

## Step 2: Verify Certificate Profiles

EJBCA uses certificate profiles to control what types of certificates a CA can issue and what algorithms are permitted. Before creating our PQC CAs, we need to verify that the built-in profiles allow ML-DSA algorithms.

### About the Default Profiles

EJBCA ships with built-in `ROOTCA` and `SUBCA` certificate profiles. In EJBCA CE, these default profiles are typically permissive — they allow all available key algorithms including ML-DSA. This means they should work out of the box for our PQC CA creation.

In production, you'd clone these defaults and create restricted profiles that only allow your chosen algorithms (e.g., only ML-DSA-87 for Root, only ML-DSA-65 for Intermediate). Keyfactor's [PQC tutorials](https://docs.keyfactor.com/ejbca/9.3.2/tutorial-build-a-post-quantum-ready-pki) walk through creating custom profiles with specific PQC algorithm restrictions — they clone `ROOTCA`, set **Available Key Algorithms** to their chosen PQC algorithm, and configure the signature algorithm explicitly.

<br>

## Step 3: Create the SassyCorp Root CA

Now we create the actual Root CA using the crypto token we just set up. The subject DN matches our OpenSSL lab exactly:

```bash
sudo bin/ejbca.sh ca init \
    --caname "SassyCorp Root CA" \
    --dn "CN=SassyCorp Root CA,OU=IT Security Division,O=SassyCorp,L=Glacier,ST=Washington,C=US,emailAddress=ca-admin@sassycorp.internal" \
    --tokenType "soft" \
    --tokenName "SassyCorp Root CA Token" \
    --keyspec "ML-DSA-87" \
    --keytype "ML-DSA" \
    -v 3650 \
    --policy null \
    -s "ML-DSA-87" \
    -certprofile "ROOTCA"
```

**What this does:**
1. Creates a new CA named "SassyCorp Root CA"
2. Uses the subject DN matching our OpenSSL lab's Root CA (including email address)
3. References the crypto token we created in Step 1 (foo123 was the pin)
4. Signs with ML-DSA-87 (quantum-resistant!)
5. 10-year validity (3650 days), same as our OpenSSL lab
6. Uses the ROOTCA certificate profile (self-signed root)

### Verify the Root CA

```bash
sudo bin/ejbca.sh ca listcas
```

**Expected Output:**
```
ManagementCA
SassyCorp Root CA
```

Get detailed info:

```bash
sudo bin/ejbca.sh ca info --caname "SassyCorp Root CA"
```

You should see the Subject DN, ML-DSA-87 as the signature algorithm, and status "Active."

<br>

## Step 4: Create the Intermediate CA Crypto Token

Same pattern as the Root CA, but with ML-DSA-65:

```bash
sudo bin/ejbca.sh cryptotoken create \
    --token "SassyCorp Intermediate CA Token" \
    --type "SoftCryptoToken" \
    --pin "foo123" \
    --autoactivate "true"
```

### Generate the ML-DSA-65 Signing Key Pair

```bash
sudo bin/ejbca.sh cryptotoken generatekey \
    --token "SassyCorp Intermediate CA Token" \
    --alias "signKey" \
    --keyspec "ML-DSA-65"
```

### Generate the RSA Encryption Key Pair

```bash
sudo bin/ejbca.sh cryptotoken generatekey \
    --token "SassyCorp Intermediate CA Token" \
    --alias "encryptKey" \
    --keyspec "4096"
```

### Verify the Token and Keys

```bash
sudo bin/ejbca.sh cryptotoken listkeys --token "SassyCorp Intermediate CA Token"
```

<br>

## Step 5: Create the SassyCorp Intermediate CA

This CA is signed by the Root CA, creating our chain of trust.  First get the Root CA's ID:

```bash
sudo bin/ejbca.sh ca info --caname "SassyCorp Root CA" | grep -i "ca id"
```

```bash
sudo bin/ejbca.sh ca init \
    --caname "SassyCorp Intermediate CA" \
    --dn "CN=SassyCorp Intermediate CA,OU=IT Security Division,O=SassyCorp,L=Glacier,ST=Washington,C=US,emailAddress=ca-admin@sassycorp.internal" \
    --tokenType "soft" \
    --tokenName "SassyCorp Intermediate CA Token" \
    --keyspec "ML-DSA-65" \
    --keytype "ML-DSA" \
    -v 1825 \
    --policy null \
    -s "ML-DSA-65" \
    -certprofile "SUBCA" \
    --signedby "****PUT ROOT CA ID HERE****"
```

**Key differences from Root CA:**

| Setting | Root CA | Intermediate CA |
|---------|---------|-----------------|
| Certificate profile | `ROOTCA` | `SUBCA` |
| Signed by | Self-signed | `SassyCorp Root CA` |
| Validity | 3650 (10 years) | 1825 (5 years) |
| Signing algorithm | ML-DSA-87 | ML-DSA-65 |
| Crypto token | SassyCorp Root CA Token | SassyCorp Intermediate CA Token |

The `--signedby "xxxxxxxxx"` is what creates the chain of trust — the Root CA signs the Intermediate CA's certificate, just like in our OpenSSL lab.

### Verify the Intermediate CA

```bash
sudo bin/ejbca.sh ca listcas
```

**Expected Output:**
```
ManagementCA
SassyCorp Root CA
SassyCorp Intermediate CA
```

<br>

## Step 6: Verify the PQC Algorithms

This is the proof that our quantum-resistant CAs are working. Export the certificates and inspect them:

### Export Root CA Certificate

```bash
sudo bin/ejbca.sh ca getcacert \
    --caname "SassyCorp Root CA" \
    -f /tmp/sassycorp-root.pem
```

### Inspect the Root CA

```bash
openssl x509 -in /tmp/sassycorp-root.pem -noout -subject -issuer
```

**Expected Output:**
```
subject=C=US, ST=Washington, L=Glacier, O=SassyCorp, OU=IT Security Division, CN=SassyCorp Root CA, emailAddress=ca-admin@sassycorp.internal
issuer=C=US, ST=Washington, L=Glacier, O=SassyCorp, OU=IT Security Division, CN=SassyCorp Root CA, emailAddress=ca-admin@sassycorp.internal
```

Root CA is self-signed (subject = issuer). Same identity as our OpenSSL lab.

### Verify ML-DSA-87 Signature

```bash
openssl x509 -in /tmp/sassycorp-root.pem -noout -text | grep "Signature Algorithm"
```

**Expected Output:**
```
    Signature Algorithm: ML-DSA-87
        Signature Algorithm: ML-DSA-87
```

### Export and Inspect the Intermediate CA

```bash
sudo bin/ejbca.sh ca getcacert \
    --caname "SassyCorp Intermediate CA" \
    -f /tmp/sassycorp-intermediate.pem
```

```bash
openssl x509 -in /tmp/sassycorp-intermediate.pem -noout -text | grep "Signature Algorithm"
```

**Expected Output:**
```
    Signature Algorithm: ML-DSA-65
        Signature Algorithm: ML-DSA-87
```

The first line is the Intermediate CA's own algorithm (ML-DSA-65). The second is the algorithm the Root CA used to sign it (ML-DSA-87). This is exactly the same pattern we saw in the OpenSSL lab.

### Verify the Certificate Chain

```bash
openssl verify -CAfile /tmp/sassycorp-root.pem /tmp/sassycorp-intermediate.pem
```

**Expected Output:**
```
/tmp/sassycorp-intermediate.pem: OK
```

The chain of trust is intact. Root CA → Intermediate CA, both quantum-resistant.

<br>

## Step 7: Restart and Health Check

Restart WildFly to ensure all the new CA state is cleanly loaded:

```bash
sudo systemctl restart wildfly-standalone
```

Wait for it to come back:

```bash
sleep 15
sudo -u wildfly /opt/wildfly/bin/jboss-cli.sh --connect ':read-attribute(name=server-state)'
```

Verify everything is operational:

```bash
curl -sk https://127.0.0.1:8442/ejbca/publicweb/healthcheck/ejbcahealth
```

**Expected Output:**
```
ALLOK
```

Clean up temp files:

```bash
sudo rm -f /tmp/sassycorp-root.pem /tmp/sassycorp-intermediate.pem
```

<br>

## What We Just Accomplished

This is a significant milestone:

```
┌──────────────────────────────────────────────────────┐
│              EJBCA CE v9.3                           │
│          3 Certificate Authorities                   │
├──────────────────────────────────────────────────────┤
│  1. ManagementCA (RSA 3072)                          │
│     └── Internal admin & TLS certificates            │
│                                                      │
│  2. SassyCorp Root CA (ML-DSA-87) ← NATIVE PQC       │
│     └── Quantum-resistant root of trust              │
│                                                      │
│  3. SassyCorp Intermediate CA (ML-DSA-65)            │
│     └── Signs end-entity certificates ← NATIVE PQC   │
│     └── Signed by SassyCorp Root CA                  │
├──────────────────────────────────────────────────────┤
│  Status: ALLOK                                       │
└──────────────────────────────────────────────────────┘
```

Your quantum-resistant CAs are now enterprise-managed. They were built natively inside EJBCA's crypto token framework with the same identity and algorithm choices as your OpenSSL lab — but now they live inside a platform that provides lifecycle management, audit logging, OCSP, CRL distribution, and web-based administration.

> **💡 Comparing the two labs:** In the OpenSSL lab, you generated keys with `openssl genpkey`, signed certificates with `openssl ca`, and managed state via flat files (`index.txt`, `serial`). Here, EJBCA's crypto tokens generate the keys, the CA subsystem handles signing, and MariaDB manages all state. Same cryptographic operations, enterprise-grade infrastructure.

<br>

## Known Limitations

A few things to be aware of as you work with PQC CAs in EJBCA:

1. **EJBCA CE `importca` does not support ML-DSA/PQC keys** — the signer code path assumes RSA. That's why we created natively.
2. **Encryption key pair must be RSA** — only the signing key can be ML-DSA. ML-KEM support for CA encryption keys isn't available yet.
3. **OCSP signing doesn't support PQC yet** — OCSP responder signing keys must use RSA or EC.
4. **`ant deploy-keystore` does NOT configure 3-port TLS** — it only creates keystores and a basic `applicationSSC` SSL context (covered in Module 04, Step 12).

<br>

## Troubleshooting

### cryptotoken create fails

Verify WildFly is running and EJBCA is deployed:

```bash
sudo systemctl status wildfly-standalone
curl -s http://127.0.0.1:8080/ejbca/ | head -5
```

### "Algorithm not recognized" or "Unknown key type"

Check that RESTEasy-Crypto was removed in Module 04 — EJBCA's Bouncy Castle (which supports ML-DSA) must be loaded instead of WildFly's older version:

```bash
ls /opt/wildfly/modules/system/layers/base/org/jboss/resteasy/resteasy-crypto/ 2>/dev/null && echo "PROBLEM: resteasy-crypto still exists!" || echo "OK: resteasy-crypto removed"
```

### ca init fails with "CA already exists"

The CA was already created. List existing CAs:

```bash
sudo bin/ejbca.sh ca listcas
```

### ca init fails with algorithm or profile error

The default certificate profiles may not include ML-DSA as an available algorithm. You'll need to set up browser access (Module 08, Steps 1-4) and create custom profiles through the Admin Web UI:

1. Navigate to **CA Functions → Certificate Profiles**
2. Clone `ROOTCA` → name it `PQCRootCA`
3. Edit: set **Available Key Algorithms** to include `ML-DSA`
4. Edit: set **Signature Algorithm** to `ML-DSA-87`
5. Save, then re-run `ca init` with `-certprofile "PQCRootCA"`

Repeat for the Intermediate CA: clone `SUBCA` → name it `PQCSubCA`, set ML-DSA-65.

### CLI switch names don't match

EJBCA CLI switch names can vary between versions. Always check:

```bash
sudo bin/ejbca.sh ca init --help
sudo bin/ejbca.sh cryptotoken generatekey --help
```

### Bouncy Castle version conflicts

Check for multiple BC jars:

```bash
find /opt/wildfly -name "bcprov*.jar" 2>/dev/null
find /opt/ejbca -name "bcprov*.jar" 2>/dev/null
```

EJBCA CE v9.3 should include Bouncy Castle 1.79+ which supports ML-DSA. If WildFly modules contain an older version, that's the conflict — revisit the RESTEasy-Crypto removal in Module 04.

### Chain verification fails

If `openssl verify` reports an error, check that both certificates were exported correctly:

```bash
openssl x509 -in /tmp/sassycorp-root.pem -noout -subject
openssl x509 -in /tmp/sassycorp-intermediate.pem -noout -issuer
```

The Intermediate CA's issuer should exactly match the Root CA's subject.

<br>

**Next Module:** [08 - Finalize Installation](08_ejbca_finalize.md)

---

*[← Previous: Module 06 - Install EJBCA](06_ejbca_install.md) | [Next: Module 08 - Finalize →](08_ejbca_finalize.md)*
