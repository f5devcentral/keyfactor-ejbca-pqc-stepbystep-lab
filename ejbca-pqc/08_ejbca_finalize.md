# Module 08: Browser Setup, Verification & Reference

## Almost There

EJBCA is running, the Management CA is active, and your PQC CAs are created. This module handles the final steps: setting up browser access to the admin UI, verifying everything works end-to-end, and providing reference material for ongoing use.

> **📋 Keyfactor Reference:** [Finalize the Installation](https://docs.keyfactor.com/ejbca-software/latest/finalize-the-installation)

<br>

## Step 1: Copy the SuperAdmin Certificate to Your Workstation

The SuperAdmin PKCS#12 file is what grants you access to EJBCA's admin interface (port 8443). You need to get this file to the machine where your web browser runs.

The file is at:

```bash
ls -la /opt/ejbca/p12/superadmin.p12
```

### If You're Working Locally (Browser on the Same Machine)

If your browser is on the same machine running EJBCA, the file is already accessible. Skip to Step 2.

### If You're Working on a Remote Server

Use `scp` to copy the file to your workstation:

```bash
scp user@ejbca-server:/opt/ejbca/p12/superadmin.p12 ~/
```

> **⚠️ Important:** The `.p12` file contains a private key. Only transfer it over encrypted connections (SCP/SFTP). Don't use unencrypted HTTP for this.

<br>

## Step 2: Set Up SSH Tunnel for Remote Access

If EJBCA is running on a remote server (VM, cloud instance, etc.), you **must** use an SSH tunnel to access the admin UI. The 3-port TLS configuration binds to `0.0.0.0`, but accessing EJBCA's mTLS port through a direct connection can be problematic. The SSH tunnel is the reliable approach:

```bash
ssh -L 8443:127.0.0.1:8443 -L 8442:127.0.0.1:8442 -L 8080:127.0.0.1:8080 user@ejbca-server
```

This forwards all three EJBCA ports to your local machine. Keep this SSH session open while you work.

After the tunnel is up, you'll access EJBCA at `https://127.0.0.1:8443/ejbca/adminweb/` from your workstation's browser — the tunnel makes it look like EJBCA is running locally.

<br>

## Step 3: Import the SuperAdmin Certificate into Your Browser

### Firefox on Ubuntu (Recommended)

Firefox on Ubuntu is the recommended browser for this lab. macOS Keychain has issues importing EJBCA-generated PKCS#12 files (see macOS note below).

1. Open **Settings** → **Privacy & Security** → scroll to **Certificates**
2. Click **View Certificates**
3. Go to the **Your Certificates** tab
4. Click **Import**
5. Select `superadmin.p12`
6. Enter the password: `ejbca`
7. Click **OK**

The `.p12` file includes the SuperAdmin certificate, its private key, AND the ManagementCA certificate chain. So you may not need to separately import the ManagementCA as a trusted authority — the chain is bundled in.

### Firefox Certificate Prompt Settings

If Firefox doesn't prompt you to select a client certificate when you visit port 8443, check this setting:

1. Open `about:config` in Firefox
2. Search for `security.default_personal_cert`
3. Set the value to `Ask Every Time`
4. **Fully restart Firefox** (not just close the tab — quit and reopen)

If Firefox *still* doesn't prompt after a full restart, the 3-port TLS configuration from Module 04, Step 12 is likely missing. Verify the `httpspriv` listener has `need-client-auth=true`:

```bash
sudo -u wildfly /opt/wildfly/bin/jboss-cli.sh --connect '/subsystem=elytron/server-ssl-context=httpspriv:read-attribute(name=need-client-auth)'
```

### Chrome/Chromium-based Browsers

1. Open **Settings** → **Privacy and Security** → **Security** → **Manage certificates**
2. Click **Import**
3. Browse to `superadmin.p12`
4. Enter the password: `ejbca`
5. Follow the wizard, accepting defaults

### macOS Note

> **⚠️ macOS Keychain can't reliably import EJBCA-generated PKCS#12 files.** The issue is an encryption format incompatibility — EJBCA/Java uses AES/PBKDF2 for PKCS#12 encryption, while macOS Keychain expects the legacy 3DES format. You may see "Unable to display information" errors or silent import failures.
>
> If you must try macOS, import via **Keychain Access GUI** (File → Import Items), not the `security import` CLI command — the CLI chokes on the format even worse. But honestly, use Firefox on Ubuntu for this lab. It just works.

<br>

## Step 4: Access the EJBCA Admin Web Interface

Open your browser and navigate to:

```
https://127.0.0.1:8443/ejbca/adminweb/
```

### What Happens

1. Your browser connects to EJBCA on port 8443 (HTTPS with mutual TLS)
2. WildFly's `httpspriv` listener requests a client certificate
3. Your browser presents the SuperAdmin certificate you just imported
4. You may be prompted to select which certificate to use — choose the SuperAdmin one
5. EJBCA validates the certificate against its Management CA
6. You're granted admin access

### Expected Result

You should see the **EJBCA Administration** page. The main dashboard shows EJBCA version information, system status, and links to CA management, enrollment, and configuration sections.

> **🎉 If you see the admin page, congratulations!** You have a fully functional EJBCA instance with quantum-resistant CAs.

<br>

## Step 5: Verify CAs in the Admin Interface

In the EJBCA Admin Web UI:

1. Navigate to **CA Functions** → **Certificate Authorities**
2. You should see three CAs listed:
   - **ManagementCA** — RSA 3072, active
   - **SassyCorp Root CA** — ML-DSA-87, active
   - **SassyCorp Intermediate CA** — ML-DSA-65, active

Click on each CA to view its details: Subject DN, Issuer DN, Key algorithm, Signature algorithm, Validity dates, and Status.

When you click on **SassyCorp Root CA**, the details should show Signature Algorithm **ML-DSA-87**. When you click on **SassyCorp Intermediate CA**, it should show Signature Algorithm **ML-DSA-65** and Signed by **SassyCorp Root CA**.

<br>

## Step 6: CLI Verification

Back on the command line, run a comprehensive check:

```bash
cd /opt/ejbca

echo "=== All CAs ==="
sudo bin/ejbca.sh ca listcas

echo ""
echo "=== EJBCA Health ==="
curl -sk https://localhost:8442/ejbca/publicweb/healthcheck/ejbcahealth

echo ""
echo "=== Root CA Certificate Check ==="
sudo bin/ejbca.sh ca getcacert --caname "SassyCorp Root CA" -f /tmp/verify-root.pem 2>/dev/null
openssl x509 -in /tmp/verify-root.pem -noout -subject -dates

echo ""
echo "=== Intermediate CA Certificate Check ==="
sudo bin/ejbca.sh ca getcacert --caname "SassyCorp Intermediate CA" -f /tmp/verify-intermediate.pem 2>/dev/null
openssl x509 -in /tmp/verify-intermediate.pem -noout -subject -dates

echo ""
echo "=== Chain Verification ==="
openssl verify -CAfile /tmp/verify-root.pem /tmp/verify-intermediate.pem
```

**Expected:** Chain verification shows `OK`.

Clean up temp files:

```bash
rm -f /tmp/verify-root.pem /tmp/verify-intermediate.pem /tmp/sassycorp-root.pem /tmp/sassycorp-intermediate.pem
```

<br>

## EJBCA URLs Quick Reference

| URL | Purpose |
|-----|---------|
| `http://localhost:8080/ejbca/` | Public web landing page (HTTP) |
| `https://localhost:8442/ejbca/` | Public web landing page (HTTPS) |
| `http://localhost:8080/ejbca/enrol/keystore.jsp` | Browser-based certificate enrollment |
| `https://localhost:8442/ejbca/publicweb/healthcheck/ejbcahealth` | Health check endpoint |
| `https://localhost:8443/ejbca/adminweb/` | Admin web interface (client cert required) |
| `https://localhost:8443/ejbca/ra/` | Registration Authority web interface |
| `https://localhost:8443/ejbca/ejbca-rest-api/v1/` | REST API |

<br>

## File Locations Reference

### EJBCA Files

| Path | Contents |
|------|----------|
| `/opt/ejbca/` | EJBCA source code and build system |
| `/opt/ejbca/conf/` | Configuration properties files |
| `/opt/ejbca/p12/` | Generated keystores and certificates |
| `/opt/ejbca/p12/superadmin.p12` | SuperAdmin client certificate |
| `/opt/ejbca/bin/ejbca.sh` | EJBCA CLI tool |
| `/opt/ejbca/doc/sql-scripts/` | Database index and setup scripts |

### WildFly Files

| Path | Contents |
|------|----------|
| `/opt/wildfly/` | Symlink to `/opt/wildfly-35.0.1.Final/` |
| `/opt/wildfly/bin/standalone.conf` | JVM and startup configuration |
| `/opt/wildfly/bin/jboss-cli.sh` | JBoss CLI tool |
| `/opt/wildfly/bin/wildfly_pass` | Elytron credential store master password |
| `/opt/wildfly/standalone/configuration/standalone.xml` | WildFly server configuration |
| `/opt/wildfly/standalone/configuration/keystore/` | TLS keystores used by WildFly |
| `/opt/wildfly/standalone/log/server.log` | WildFly application log |
| `/opt/wildfly/standalone/deployments/ejbca.ear` | Deployed EJBCA application |

### System Configuration

| Path | Contents |
|------|----------|
| `/etc/profile.d/java.sh` | JAVA_HOME environment variable |
| `/etc/profile.d/ejbca.sh` | EJBCA_HOME and APPSRV_HOME variables |
| `/etc/systemd/system/wildfly-standalone.service` | WildFly systemd service |
| `/etc/mysql/mariadb.conf.d/50-server.cnf` | MariaDB server configuration |

<br>

## Network Ports Reference

| Port | Protocol | Authentication | Purpose |
|------|----------|----------------|---------|
| 3306 | TCP | Password | MariaDB (localhost only) |
| 4447 | TCP | — | JBoss Remoting (EJBCA CLI) |
| 8080 | HTTP | None | Public web pages, health check |
| 8442 | HTTPS | Server cert only | Public HTTPS access |
| 8443 | HTTPS | Mutual TLS | Admin web interface (client cert required) |
| 9990 | HTTP | — | WildFly management console (localhost) |

---

## Passwords Reference (Lab Values)

> **⚠️ These are lab passwords. Never use these in production.**

| Component | Username | Password | Purpose |
|-----------|----------|----------|---------|
| MariaDB root | `root` | (set in Module 03) | Database administration |
| MariaDB EJBCA | `ejbca` | `ejbca` | EJBCA database access |
| SuperAdmin P12 | — | `ejbca` | Admin browser certificate |
| TLS keystore | — | `serverpwd` | WildFly TLS |
| Trust store | — | `changeit` | Java trust store |
| PQC CA crypto tokens | — | `foo123` | PQC CA activation PINs |
| Credential store | — | (random, in wildfly_pass) | Elytron encrypted passwords |

<br>

## Service Management Quick Reference

```bash
# WildFly (EJBCA runs inside WildFly)
sudo systemctl start wildfly-standalone
sudo systemctl stop wildfly-standalone
sudo systemctl restart wildfly-standalone
sudo systemctl status wildfly-standalone

# MariaDB
sudo systemctl start mariadb
sudo systemctl stop mariadb
sudo systemctl restart mariadb
sudo systemctl status mariadb
```

### EJBCA CLI Quick Reference

```bash
cd /opt/ejbca

# List all CAs
sudo bin/ejbca.sh ca listcas

# Get CA info
sudo bin/ejbca.sh ca info --caname "SassyCorp Root CA"

# Export CA certificate
sudo bin/ejbca.sh ca getcacert --caname "SassyCorp Root CA" -f /tmp/root-ca.pem

# List roles
sudo bin/ejbca.sh roles listroles

# List admins in a role
sudo bin/ejbca.sh roles listadmins --role "Super Administrator Role"

# Full rebuild and deploy
ant -q clean deployear

# Deploy TLS keystores
ant deploy-keystore

# Run installer (initial setup only)
ant runinstall
```

### JBoss CLI Quick Reference

```bash
# Check server state
sudo -u wildfly /opt/wildfly/bin/jboss-cli.sh --connect ':read-attribute(name=server-state)'

# Test datasource connection
sudo -u wildfly /opt/wildfly/bin/jboss-cli.sh --connect '/subsystem=datasources/data-source=ejbcads:test-connection-in-pool'

# Check deployment status
sudo -u wildfly /opt/wildfly/bin/jboss-cli.sh --connect '/deployment=ejbca.ear:read-attribute(name=status)'

# Reload configuration
sudo -u wildfly /opt/wildfly/bin/jboss-cli.sh --connect ':reload'

# View credential store aliases
sudo -u wildfly /opt/wildfly/bin/jboss-cli.sh --connect '/subsystem=elytron/credential-store=defaultCS:read-aliases'
```

<br>

## Full System Health Check Script

Run this to verify the entire stack:

```bash
echo "======================================"
echo "  EJBCA + PQC CA System Health Check"
echo "======================================"
echo ""

echo "--- Services ---"
sudo systemctl is-active mariadb && echo "MariaDB:  ✅ RUNNING" || echo "MariaDB:  ❌ STOPPED"
sudo systemctl is-active wildfly-standalone && echo "WildFly:  ✅ RUNNING" || echo "WildFly:  ❌ STOPPED"

echo ""
echo "--- Ports ---"
sudo ss -tlnp | grep -q ":3306 " && echo "3306 (MariaDB): ✅ LISTENING" || echo "3306 (MariaDB): ❌ NOT LISTENING"
sudo ss -tlnp | grep -q ":8080 " && echo "8080 (HTTP):    ✅ LISTENING" || echo "8080 (HTTP):    ❌ NOT LISTENING"
sudo ss -tlnp | grep -q ":8442 " && echo "8442 (HTTPS):   ✅ LISTENING" || echo "8442 (HTTPS):   ❌ NOT LISTENING"
sudo ss -tlnp | grep -q ":8443 " && echo "8443 (mTLS):    ✅ LISTENING" || echo "8443 (mTLS):    ❌ NOT LISTENING"

echo ""
echo "--- Database ---"
mariadb -u ejbca -pejbca -h 127.0.0.1 -e "SELECT 1" ejbca >/dev/null 2>&1 && echo "DB Connection:  ✅ OK" || echo "DB Connection:  ❌ FAILED"

echo ""
echo "--- EJBCA Health ---"
HEALTH=$(curl -sk https://localhost:8442/ejbca/publicweb/healthcheck/ejbcahealth 2>/dev/null)
[ "$HEALTH" = "ALLOK" ] && echo "EJBCA Health:   ✅ $HEALTH" || echo "EJBCA Health:   ❌ $HEALTH"

echo ""
echo "--- Certificate Authorities ---"
cd /opt/ejbca 2>/dev/null && sudo bin/ejbca.sh ca listcas 2>/dev/null || echo "Could not list CAs"

echo ""
echo "--- PQC Algorithm Verification ---"
cd /opt/ejbca 2>/dev/null
sudo bin/ejbca.sh ca getcacert --caname "SassyCorp Root CA" -f /tmp/health-root.pem 2>/dev/null
if [ -f /tmp/health-root.pem ]; then
    ALGO=$(openssl x509 -in /tmp/health-root.pem -noout -text 2>/dev/null | grep "Signature Algorithm" | head -1 | awk '{print $NF}')
    echo "Root CA Algo:      ✅ $ALGO"
    rm -f /tmp/health-root.pem
else
    echo "Root CA Algo:      ❌ Could not verify"
fi

sudo bin/ejbca.sh ca getcacert --caname "SassyCorp Intermediate CA" -f /tmp/health-int.pem 2>/dev/null
if [ -f /tmp/health-int.pem ]; then
    ALGO=$(openssl x509 -in /tmp/health-int.pem -noout -text 2>/dev/null | grep "Signature Algorithm" | head -1 | awk '{print $NF}')
    echo "Intermediate Algo: ✅ $ALGO"
    rm -f /tmp/health-int.pem
else
    echo "Intermediate Algo: ❌ Could not verify"
fi

echo ""
echo "======================================"
```

<br>

## What You've Accomplished

Let's take a step back and appreciate the full journey:

1. **Lab 1 (FIPS PQC):** You built a quantum-resistant CA from scratch using OpenSSL — understanding PQC algorithms, key generation, certificate signing, and chain-of-trust at the deepest level.

2. **Lab 2 (This Lab):** You took that knowledge and built enterprise-grade PQC CAs natively inside an enterprise PKI management platform — learning Java application servers, database configuration, build systems, and enterprise CA management along the way.

Your SassyCorp PKI now has:
- **Quantum-resistant algorithms** (ML-DSA-87 Root, ML-DSA-65 Intermediate)
- **Enterprise management** (EJBCA web UI, CLI, REST API)
- **Certificate lifecycle support** (issuance, renewal, revocation)
- **Audit logging** (all operations tracked in MariaDB)
- **Protocol support** (SCEP, CMP, OCSP, CRL)
- **Role-based access control** (SuperAdmin role with certificate authentication)

Not bad for a learning lab.

---

## Next Steps (Beyond This Lab)

If you want to keep exploring, here are some things you can do with your EJBCA deployment:

- **Issue Quantum Secure end-entity certificates** using the Intermediate CA through EJBCA's enrollment interface
- **Configure CRL distribution points** for your PQC CAs
- **Set up OCSP** for real-time certificate status checking (note: OCSP signing keys must be RSA/EC for now)
- **Create certificate profiles** tailored for different use cases (TLS server, code signing, email)
- **Explore the REST API** at `https://127.0.0.1:8443/ejbca/ejbca-rest-api/v1/`
- **Test hybrid certificates** combining PQC + traditional algorithms (EJBCA supports this)

---

## Architecture Diagram — Final State

```
┌─────────────────────────────────────────────────────────────────┐
│                         Your Browser                            │
│  SuperAdmin.p12 (client cert) ─────────────────────-─┐          │
└──────────────────────────────────────────────────────┼──────────┘
                                                       │
                                               SSH Tunnel (8443)
                                                       │
                                                       ▼
┌─────────────────────────────────────────────────────────────────┐
│                    Ubuntu 25.10 Server                          │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ┌───────────────────────────────────────────────────────────┐  │
│  │              WildFly 35.0.1.Final                         │  │
│  │     Port 8080 (HTTP) │ 8442 (HTTPS) │ 8443 (mTLS)         │  │
│  │  ┌─────────────────────────────────────────────────────┐  │  │
│  │  │            EJBCA Community Edition v9.3             │  │  │
│  │  │                                                     │  │  │
│  │  │  ┌──────────────┐ ┌────────────────────────────┐    │  │  │
│  │  │  │ ManagementCA │ │ SassyCorp Root CA          │    │  │  │
│  │  │  │ (RSA 3072)   │ │ (ML-DSA-87) ← Native PQC   │    │  │  │
│  │  │  └──────────────┘ └──────────────┬─────────────┘    │  │  │
│  │  │                                  │                  │  │  │
│  │  │                   ┌──────────────▼─────────────┐    │  │  │
│  │  │                   │ SassyCorp Intermediate CA  │    │  │  │
│  │  │                   │ (ML-DSA-65) ← Native PQC   │    │  │  │
│  │  │                   └────────────────────────────┘    │  │  │
│  │  └─────────────────────────────────────────────────────┘  │  │
│  └───────────────────────────────────────────────────────────┘  │
│                              │                                  │
│                              ▼                                  │
│  ┌───────────────────────────────────────────────────────────┐  │
│  │              MariaDB (port 3306)                          │  │
│  │              Database: ejbca                              │  │
│  └───────────────────────────────────────────────────────────┘  │
│                                                                 │
│  ┌───────────────────────────────────────────────────────────┐  │
│  │  Original PQC Files (read-only reference):                │  │
│  │  /opt/sassycorp-pqc/root-ca/                              │  │
│  │  /opt/sassycorp-pqc/intermediate-ca/                      │  │
│  └───────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────┘
```

<br>

## Troubleshooting

### Browser doesn't prompt for client certificate

1. Check `about:config` → `security.default_personal_cert` = `Ask Every Time`
2. Fully restart Firefox (quit and reopen, not just the tab)
3. Verify 3-port TLS config: `sudo -u wildfly /opt/wildfly/bin/jboss-cli.sh --connect '/subsystem=elytron/server-ssl-context=httpspriv:read-attribute(name=need-client-auth)'`
4. Verify port 8443 is listening: `sudo ss -tlnp | grep 8443`

### "Not authorized" error in admin web

The SuperAdmin certificate may not be properly linked to the admin role:

```bash
cd /opt/ejbca
sudo bin/ejbca.sh roles listroles
sudo bin/ejbca.sh roles listadmins --role "Super Administrator Role"
```

### Certificate import fails in browser

Verify the `.p12` file isn't corrupted:

```bash
openssl pkcs12 -in /opt/ejbca/p12/superadmin.p12 -passin pass:ejbca -nokeys -clcerts | openssl x509 -noout -subject
```

### Need to regenerate the SuperAdmin certificate

```bash
cd /opt/ejbca
ant deploy-keystore
sudo systemctl restart wildfly-standalone
```

Then re-import the new `superadmin.p12` into your browser.

### EJBCA won't start after reboot

Check startup order — MariaDB must start before WildFly:

```bash
sudo systemctl status mariadb
sudo systemctl status wildfly-standalone
```

If WildFly started before MariaDB was ready, restart it:

```bash
sudo systemctl restart wildfly-standalone
```

### Database connection lost

```bash
# Check MariaDB is running
sudo systemctl status mariadb

# Test TCP connection (what WildFly uses)
mariadb -u ejbca -p -h 127.0.0.1 -e "SELECT 1" ejbca

# Test WildFly datasource
sudo -u wildfly /opt/wildfly/bin/jboss-cli.sh --connect '/subsystem=datasources/data-source=ejbcads:test-connection-in-pool'
```

### Bouncy Castle / PQC errors

```bash
# Check for multiple BC jars
find /opt/wildfly -name "bcprov*.jar" 2>/dev/null

# Check the RESTEasy-Crypto removal
ls /opt/wildfly/modules/system/layers/base/org/jboss/resteasy/resteasy-crypto/ 2>/dev/null
echo $?  # Should return 2 (directory doesn't exist)
```

### Environment variables missing after reboot

```bash
source /etc/profile.d/java.sh
source /etc/profile.d/ejbca.sh
echo "JAVA_HOME=$JAVA_HOME"
echo "EJBCA_HOME=$EJBCA_HOME"
echo "APPSRV_HOME=$APPSRV_HOME"
```

---

## References

| Resource | URL |
|----------|-----|
| EJBCA CE Source | https://github.com/Keyfactor/ejbca-ce |
| EJBCA Documentation | https://docs.keyfactor.com/ejbca-software/latest/installation |
| EJBCA Security Guide | https://docs.keyfactor.com/ejbca/latest/ejbca-security |
| EJBCA REST API | https://docs.keyfactor.com/ejbca/latest/ejbca-rest-interface |
| WildFly 35 Documentation | https://docs.wildfly.org/35/ |
| MariaDB Documentation | https://mariadb.com/kb/en/documentation/ |
| OpenSSL PQC Lab | https://github.com/f5devcentral/openssl-pqc-stepbystep-lab/tree/main/fipsqs |
| NIST FIPS 204 (ML-DSA) | https://csrc.nist.gov/pubs/fips/204/final |
| Keyfactor PQC Hybrid Tutorial | https://docs.keyfactor.com/ejbca/9.0/tutorial-create-pqc-hybrid-ca-chain |

---

*This lab is part of the [Post-Quantum Cryptography Step-by-Step Lab](https://github.com/f5devcentral/openssl-pqc-stepbystep-lab) series. For educational and internal testing purposes. Production deployments should use HSMs, air-gapped Root CAs, and follow organizational security policies.*

---

*[← Previous: Module 07 - Create PQC CAs](07_ejbca_create_pqc_ca.md) | [Back to Introduction →](00_ejbca_migration_intro.md)*
