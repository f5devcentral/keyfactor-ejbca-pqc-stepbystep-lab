# Module 06: Install EJBCA Software Stack

## What Happens During Installation?

With EJBCA deployed and running in WildFly, we now need to *initialize* it as a Certificate Authority. This is where `ant runinstall` does its magic:

1. **Creates the Management CA** — an internal RSA CA that signs admin certificates and the TLS server certificate
2. **Generates the TLS keystore** — so EJBCA can serve HTTPS on port 8443
3. **Creates the SuperAdmin certificate** — a PKCS#12 client certificate you'll import into your browser to access the admin UI
4. **Configures WildFly TLS** — installs the keystore into WildFly's Elytron security subsystem

After this module, you'll have a fully functional EJBCA instance with web-based administration. Module 07 then creates our PQC CAs natively on top of this foundation.

> **📋 Keyfactor Reference:** [Install EJBCA Software Stack](https://docs.keyfactor.com/ejbca-software/latest/install-ejbca-software-stack)

---

## Step 1: Run the EJBCA Installer

```bash
cd /opt/ejbca
sudo ant runinstall
```

**What this does behind the scenes:**
1. Reads `install.properties` to determine the Management CA configuration
2. Creates a crypto token (soft key store) for the Management CA
3. Generates an RSA 3072-bit key pair for the Management CA
4. Creates a self-signed Root CA certificate for the Management CA
5. Stores everything in the MariaDB database

**Watch the output carefully.** You should see:

```
[echo] Initializing CA with:
[echo]     ca.name = ManagementCA
[echo]     ca.dn = CN=ManagementCA,O=SassyCorp,C=US
...
BUILD SUCCESSFUL
```

If you see `BUILD FAILED`, check:
- Is WildFly still running? (`sudo systemctl status wildfly-standalone`)
- Is the EJBCA EAR deployed? (`curl -s http://localhost:8080/ejbca/`)
- Are the properties files correct? (Especially `install.properties`)

> **💡 Patience:** This command creates cryptographic keys and certificates, which involves entropy gathering. On a VM with limited entropy, this could take a minute or two. If it seems hung, it's probably waiting for random data. You can run `cat /proc/sys/kernel/random/entropy_avail` in another terminal to check — values above 256 are sufficient.

---

## Step 2: Deploy the TLS Keystore to WildFly

Now we tell EJBCA to generate the server TLS certificate and configure WildFly to use it:

```bash
sudo ant deploy-keystore
sudo chown -R $(whoami):$(whoami) /opt/ejbca
sudo chown -R wildfly:wildfly /opt/wildfly/standalone/configuration/keystore/
```

> **💡 Why `sudo`?** Both ant runinstall and deploy-keystore require sudo — runinstall writes build artifacts that need elevated permissions, and deploy-keystore copies keystores into /opt/wildfly/standalone/configuration/keystore/, owned by the wildfly user. The chown commands afterward restore proper ownership: your user owns /opt/ejbca and the wildfly user owns the keystore directory.

**What this does:**
1. Generates a TLS server certificate signed by the Management CA
2. Creates the server keystore at `/opt/ejbca/p12/keystore.p12` (PKCS#12 format)
3. Creates the trust store at `/opt/ejbca/p12/truststore.p12` (PKCS#12 format)
4. Copies both keystores to `/opt/wildfly/standalone/configuration/keystore/`

> **💡 Note:** `ant deploy-keystore` creates the keystore *files* and copies them to WildFly — but it does **not** configure WildFly's Elytron TLS subsystem. That was already done in Module 04, Step 12, where we set up the key stores, SSL contexts, and Undertow listeners pointing to these file paths. The two steps work together: Module 04 tells WildFly *where to look*, Module 06 *puts the files there*.

**Expected Output:**
```
BUILD SUCCESSFUL
```

### Verify the Keystores Were Created

```bash
ls -la /opt/ejbca/p12/
```

You should see several `.p12` files:

| File | Purpose |
|------|---------|
| `tomcat.p12` | TLS server keystore (copied to WildFly as keystore.p12) |
| `truststore.p12` | Trust store containing the Management CA certificate |
| `superadmin.p12` | SuperAdmin client certificate (for browser import) |

<br>

### Reload the Elytron Keystores

The TLS keystores didn't exist when WildFly first loaded the Elytron SSL contexts back in Module 04 — so WildFly's trust manager has no certificates loaded yet. We need to tell Elytron to reload them now that the files are in place:
```bash
sudo -u wildfly /opt/wildfly/bin/jboss-cli.sh --connect '/subsystem=elytron/key-store=httpsKS:load()'
sudo -u wildfly /opt/wildfly/bin/jboss-cli.sh --connect '/subsystem=elytron/key-store=httpsTS:load()'
```

**Expected Output:** Both should return `{"outcome" => "success"}`.

> **💡 Why this matters:** Without this step, port 8443's mutual TLS won't enforce client certificate authentication — the Elytron SSL context has `need-client-auth=true`, but with an empty trust store it has nothing to validate against and silently allows all connections. Reloading the keystores activates the trust manager with the Management CA certificate, so WildFly will properly demand a client certificate on port 8443.

## Step 3: Restart WildFly

WildFly needs to restart to pick up the new TLS keystores:

```bash
sudo systemctl restart wildfly-standalone
```

Wait for it to fully start:

```bash
sleep 15
sudo -u wildfly /opt/wildfly/bin/jboss-cli.sh --connect ':read-attribute(name=server-state)'
```

**Expected:** `"running"`

---

## Step 4: Verify HTTPS is Working

Test the public HTTPS port (8442 — no client certificate required):

```bash
curl -sk https://127.0.0.1:8442/ejbca/publicweb/healthcheck/ejbcahealth
```

**Expected Output:**
```
ALLOK
```

If you see `ALLOK`, EJBCA is running, connected to the database, and serving HTTPS. That's a big milestone.

Test the admin HTTPS port (8443 — requires client certificate):

```bash
curl -sk https://127.0.0.1:8443/ejbca/adminweb/
```

This will likely return an error or empty response because we haven't provided a client certificate. That's correct — port 8443 requires mutual TLS authentication.

---

## Step 5: Locate the SuperAdmin Certificate

The `ant deploy-keystore` command generated a SuperAdmin PKCS#12 file:

```bash
ls -la /opt/ejbca/p12/superadmin.p12
```

This file contains:
- The SuperAdmin's private key
- The SuperAdmin's certificate (signed by the Management CA)
- The Management CA certificate

The password for this PKCS#12 file is what you set in `web.properties`:

```
superadmin.password=ejbca
```

### View the SuperAdmin Certificate Details

```bash
openssl pkcs12 -in /opt/ejbca/p12/superadmin.p12 -passin pass:ejbca -nokeys -clcerts | openssl x509 -noout -subject -issuer
```

**Expected Output:**
```
subject=CN=SuperAdmin, O=SassyCorp, C=US
issuer=CN=ManagementCA, O=SassyCorp, C=US
```

> **💡 Browser Import:** You'll import this `.p12` file into your browser in Module 08 to access the admin web UI. For now, just note where it is.

---

## Step 6: Verify the Management CA

Let's confirm the Management CA was created in the database:

```bash
cd /opt/ejbca
sudo bin/ejbca.sh ca listcas
```

**Expected Output:**
```
ManagementCA
```

Get more details:

```bash
sudo bin/ejbca.sh ca info --caname "ManagementCA"
```

This should show the Management CA's Subject DN, key algorithm (RSA 3072), validity, and status.

---

## What We Just Built

Here's the state of our EJBCA installation after this module:

```
┌──────────────────────────────────────────────┐
│              EJBCA CE v9.3                   │
│  Status: Running & Initialized               │
├──────────────────────────────────────────────┤
│  Management CA: ManagementCA                 │
│    DN: CN=ManagementCA,O=SassyCorp,C=US      │
│    Key: RSA 3072-bit (soft token)            │
│    Algo: SHA256WithRSA                       │
│    Purpose: Admin TLS, SuperAdmin certs      │
├──────────────────────────────────────────────┤
│  TLS Configuration:                          │
│    Port 8080: HTTP (public)                  │
│    Port 8442: HTTPS (public, no client cert) │
│    Port 8443: HTTPS (admin, client cert req) │
├──────────────────────────────────────────────┤
│  SuperAdmin: CN=SuperAdmin,O=SassyCorp,C=US  │
│    File: /opt/ejbca/p12/superadmin.p12       │
│    Password: ejbca                           │
├──────────────────────────────────────────────┤
│  PQC CAs: NOT YET CREATED (Module 07)        │
└──────────────────────────────────────────────┘
```

The Management CA is EJBCA's internal plumbing — it keeps the admin interface secure. Our actual PQC certificate authorities are coming next.

---

## Troubleshooting

### ant runinstall fails with "CA already exists"

If you're re-running the installer, the Management CA may already exist. This is fine — it means `runinstall` was already successful.

### HTTPS health check returns connection refused

WildFly may not have finished starting. Wait 30 seconds and try again. Also check:

```bash
sudo ss -tlnp | grep -E "8080|8442|8443"
```

All three ports should be listening.

### "No CA certificate" or keystore errors

The keystores may not have been deployed correctly. Re-run:

```bash
cd /opt/ejbca
sudo ant deploy-keystore
sudo systemctl restart wildfly-standalone
```

### ejbca.sh command not found

Make sure you're in the EJBCA directory:

```bash
cd /opt/ejbca
bin/ejbca.sh ca listcas
```

The `ejbca.sh` script is in the `bin/` directory under `EJBCA_HOME`.

---

**Next Module:** [07 - Create PQC CAs](07_ejbca_create_pqc_ca.md)

---

*[← Previous: Module 05 - Deploy EJBCA](05_ejbca_deploy.md) | [Next: Module 07 - Create PQC CAs →](07_ejbca_create_pqc_ca.md)*
