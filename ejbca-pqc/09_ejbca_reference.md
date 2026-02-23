# Module 09: Deployment Reference & Troubleshooting

## Complete Reference Guide

This module serves as your ongoing reference for the EJBCA deployment. Bookmark it. You'll come back to it when you need to remember a file path, a port number, or how to fix something.

> **📋 Keyfactor Reference:** [Deployment Reference](https://docs.keyfactor.com/ejbca-software/latest/deployment-reference) and [EJBCA Security](https://docs.keyfactor.com/ejbca/latest/ejbca-security)

---

## File Locations

### EJBCA Files

| Path | Contents |
|------|----------|
| `/opt/ejbca/` | EJBCA source code and build system |
| `/opt/ejbca/conf/` | Configuration properties files |
| `/opt/ejbca/conf/install.properties` | Management CA configuration |
| `/opt/ejbca/conf/cesecore.properties` | Core security settings |
| `/opt/ejbca/conf/ejbca.properties` | Application settings |
| `/opt/ejbca/conf/web.properties` | Web interface and admin settings |
| `/opt/ejbca/conf/database.properties` | Database connection settings |
| `/opt/ejbca/p12/` | Generated keystores and certificates |
| `/opt/ejbca/p12/keystore.p12` | WildFly TLS server keystore |
| `/opt/ejbca/p12/truststore.p12` | Trust store (Management CA cert) |
| `/opt/ejbca/p12/superadmin.p12` | SuperAdmin client certificate |
| `/opt/ejbca/bin/ejbca.sh` | EJBCA CLI tool |
| `/opt/ejbca/doc/sql-scripts/` | Database index and setup scripts |

### WildFly Files

| Path | Contents |
|------|----------|
| `/opt/wildfly/` | Symlink to WildFly installation |
| `/opt/wildfly-35.0.1.Final/` | Actual WildFly installation directory |
| `/opt/wildfly/bin/standalone.conf` | JVM and startup configuration |
| `/opt/wildfly/bin/jboss-cli.sh` | JBoss CLI tool |
| `/opt/wildfly/bin/wildfly_pass` | Elytron credential store master password |
| `/opt/wildfly/standalone/configuration/standalone.xml` | WildFly server configuration |
| `/opt/wildfly/standalone/configuration/keystore/` | TLS keystores used by WildFly |
| `/opt/wildfly/standalone/log/server.log` | WildFly application log |
| `/opt/wildfly/standalone/deployments/ejbca.ear` | Deployed EJBCA application |
| `/opt/wildfly/standalone/deployments/mariadb-java-client.jar` | MariaDB JDBC driver |

### Database Files

| Path | Contents |
|------|----------|
| `/etc/mysql/mariadb.conf.d/50-server.cnf` | MariaDB server configuration |
| `/var/lib/mysql/` | MariaDB data directory |
| `/var/log/mysql/` | MariaDB logs |

### System Configuration

| Path | Contents |
|------|----------|
| `/etc/profile.d/java.sh` | JAVA_HOME environment variable |
| `/etc/profile.d/ejbca.sh` | EJBCA_HOME and APPSRV_HOME variables |
| `/etc/systemd/system/wildfly-standalone.service` | WildFly systemd service |
| `/etc/sysconfig/wildfly-standalone.conf` | WildFly service configuration |

---

## Network Ports

| Port | Protocol | Authentication | Purpose |
|------|----------|----------------|---------|
| 3306 | TCP | Password | MariaDB (localhost only) |
| 4447 | TCP | — | JBoss Remoting (EJBCA CLI) |
| 8080 | HTTP | None | Public web pages, health check |
| 8442 | HTTPS | Server cert only | Public HTTPS access |
| 8443 | HTTPS | Mutual TLS | Admin web interface (client cert required) |
| 9990 | HTTP | — | WildFly management console (localhost) |

---

## Service Management

### Start/Stop/Restart Services

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

### Check If Services Are Running

```bash
sudo systemctl is-active wildfly-standalone && echo "WildFly: RUNNING" || echo "WildFly: STOPPED"
sudo systemctl is-active mariadb && echo "MariaDB: RUNNING" || echo "MariaDB: STOPPED"
```

### View Logs

```bash
# WildFly systemd journal
sudo journalctl -u wildfly-standalone --no-pager -n 100

# WildFly server log
sudo tail -100 /opt/wildfly/standalone/log/server.log

# Follow WildFly log in real-time
sudo tail -f /opt/wildfly/standalone/log/server.log

# MariaDB log
sudo journalctl -u mariadb --no-pager -n 100
```

---

## EJBCA CLI Quick Reference

All commands run from `/opt/ejbca`:

```bash
cd /opt/ejbca
```

### CA Operations

```bash
# List all CAs
bin/ejbca.sh ca listcas

# Get CA info
bin/ejbca.sh ca info --caname "SassyCorp Root CA"

# Export CA certificate
bin/ejbca.sh ca getcacert --caname "SassyCorp Root CA" -f /tmp/root-ca.pem
```

### Crypto Token Operations

```bash
# List crypto tokens
bin/ejbca.sh cryptotoken list

# Generate a key pair in a crypto token
bin/ejbca.sh cryptotoken generatekey --token "TokenName" --alias "keyAlias" --keyspec "ML-DSA-87"

# Create a new crypto token
bin/ejbca.sh cryptotoken create --token "TokenName" --pin "foo123" --type "SoftCryptoToken" --autoactivate true
```

### CA Initialization (Native Creation)

```bash
# Initialize a new CA using a crypto token
bin/ejbca.sh ca init \
  --caname "CA Name" \
  --dn "CN=CA Name,O=Org,C=US" \
  --tokenType "soft" \
  --tokenName "TokenName" \
  --keyspec "ML-DSA-87" \
  --keytype "ML-DSA" \
  -v 3650 \
  --policy null \
  -s "ML-DSA-87" \
  -certprofile "ROOTCA"
```

### Role Operations

```bash
# List roles
bin/ejbca.sh roles listroles

# List admins in a role
bin/ejbca.sh roles listadmins "Super Administrator Role"
```

### Build and Deploy

```bash
# Full rebuild and deploy
sudo ant -q clean deployear

# Deploy TLS keystores
sudo ant deploy-keystore

# Run installer (initial setup only)
ant runinstall
```

---

## JBoss CLI Quick Reference

```bash
# Connect to running WildFly
sudo -u wildfly /opt/wildfly/bin/jboss-cli.sh --connect

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

# Verify SSL context mutual TLS
sudo -u wildfly /opt/wildfly/bin/jboss-cli.sh --connect '/subsystem=elytron/server-ssl-context=httpspriv:read-attribute(name=need-client-auth)'
```

---

## Database Quick Reference

```bash
# Connect to EJBCA database
mariadb -u ejbca -p ejbca

# Check EJBCA tables
mariadb -u ejbca -p -e "USE ejbca; SHOW TABLES;" | head -20

# Check CA data in database
mariadb -u ejbca -p -e "USE ejbca; SELECT name, status FROM CAData;"

# Check database size
mariadb -u ejbca -p -e "SELECT table_schema, ROUND(SUM(data_length + index_length) / 1024 / 1024, 2) AS 'Size (MB)' FROM information_schema.tables WHERE table_schema = 'ejbca' GROUP BY table_schema;"
```

---

## Environment Variables

```bash
# Required environment variables
echo "JAVA_HOME=$JAVA_HOME"
echo "EJBCA_HOME=$EJBCA_HOME"
echo "APPSRV_HOME=$APPSRV_HOME"
```

If any are missing, reload them:

```bash
source /etc/profile.d/java.sh
source /etc/profile.d/ejbca.sh
```

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
| PQC crypto tokens | — | `foo123` | PQC CA soft token PIN |
| Credential store | — | (random, in `/opt/wildfly/bin/wildfly_pass`) | Elytron encrypted passwords |

---

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
sudo ss -tlnp | grep -q ":8443 " && echo "8443 (Admin):   ✅ LISTENING" || echo "8443 (Admin):   ❌ NOT LISTENING"

echo ""
echo "--- Database ---"
mariadb -u ejbca -pejbca -e "SELECT 1" ejbca >/dev/null 2>&1 && echo "DB Connection:  ✅ OK" || echo "DB Connection:  ❌ FAILED"

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
    echo "Root CA Algo:      ❌ Could not verify (CA may not be created yet)"
fi

sudo bin/ejbca.sh ca getcacert --caname "SassyCorp Intermediate CA" -f /tmp/health-int.pem 2>/dev/null
if [ -f /tmp/health-int.pem ]; then
    ALGO=$(openssl x509 -in /tmp/health-int.pem -noout -text 2>/dev/null | grep "Signature Algorithm" | head -1 | awk '{print $NF}')
    echo "Intermediate Algo: ✅ $ALGO"
    rm -f /tmp/health-int.pem
else
    echo "Intermediate Algo: ❌ Could not verify (CA may not be created yet)"
fi

echo ""
echo "======================================"
```

---

## Common Troubleshooting Scenarios

### EJBCA Won't Start After Reboot

Check startup order — MariaDB must start before WildFly:

```bash
sudo systemctl status mariadb
sudo systemctl status wildfly-standalone
```

If WildFly started before MariaDB was ready, restart it:

```bash
sudo systemctl restart wildfly-standalone
```

### WildFly Silently Fails to Start

Check if it's the launch.sh shebang issue (Ubuntu uses dash for `/bin/sh`):

```bash
head -1 /opt/wildfly/bin/systemd/launch.sh
```

If it shows `#!/bin/sh`, fix it:

```bash
sudo sed -i '1s|#!/bin/sh|#!/bin/bash|' /opt/wildfly/bin/systemd/launch.sh
sudo systemctl restart wildfly-standalone
```

Also verify `/etc/sysconfig/wildfly-standalone.conf` has its variables uncommented (WILDFLY_SH, WILDFLY_SERVER_CONFIG, WILDFLY_BIND, WILDFLY_CONSOLE_LOG, JAVA_HOME).

### Out of Memory Errors

Check the heap setting in `/opt/wildfly/bin/standalone.conf`:

```bash
grep -i "xmx" /opt/wildfly/bin/standalone.conf
```

Should show `-Xmx2048m` (2 GB). Increase if needed:

```bash
sudo sed -i 's/-Xmx2048m/-Xmx4096m/' /opt/wildfly/bin/standalone.conf
sudo sed -i 's/-Xms2048m/-Xms4096m/' /opt/wildfly/bin/standalone.conf
sudo systemctl restart wildfly-standalone
```

### Database Connection Lost

```bash
# Check MariaDB is running
sudo systemctl status mariadb

# Test connection (use TCP like WildFly does)
mariadb -u ejbca -p -h 127.0.0.1 -e "SELECT 1;" ejbca

# Test WildFly datasource
sudo -u wildfly /opt/wildfly/bin/jboss-cli.sh --connect '/subsystem=datasources/data-source=ejbcads:test-connection-in-pool'
```

### Bouncy Castle / PQC Errors

The most common PQC-related issue is Bouncy Castle version conflicts:

```bash
# Check for multiple BC jars
find /opt/wildfly -name "bcprov*.jar" 2>/dev/null
find /opt/wildfly -name "bcpkix*.jar" 2>/dev/null

# Check the RESTEasy-Crypto removal
ls /opt/wildfly/modules/system/layers/base/org/jboss/resteasy/resteasy-crypto/ 2>/dev/null
echo $?  # Should return 2 (directory doesn't exist)
```

### Admin UI Shows "Not Authorized"

1. Verify the SuperAdmin certificate is imported in your browser
2. Verify the Management CA is trusted in your browser
3. Check the admin role mapping:

```bash
cd /opt/ejbca
bin/ejbca.sh roles listroles
bin/ejbca.sh roles listadmins "Super Administrator Role"
```

### Port 8443 Not Requiring Client Certificate

Verify the httpspriv SSL context enforces mutual TLS:

```bash
sudo -u wildfly /opt/wildfly/bin/jboss-cli.sh --connect '/subsystem=elytron/server-ssl-context=httpspriv:read-attribute(name=need-client-auth)'
```

Should return `true`. If not:

```bash
sudo -u wildfly /opt/wildfly/bin/jboss-cli.sh --connect '/subsystem=elytron/server-ssl-context=httpspriv:write-attribute(name=need-client-auth,value=true)'
sudo -u wildfly /opt/wildfly/bin/jboss-cli.sh --connect ':reload'
```

### Need to Regenerate the SuperAdmin Certificate

```bash
cd /opt/ejbca
sudo ant deploy-keystore
sudo systemctl restart wildfly-standalone
```

Then re-import the new `superadmin.p12` into your browser.

---

## Architecture Diagram — Final State

```
┌─────────────────────────────────────────────────────────────────┐
│                         Your Browser                            │
│  SuperAdmin.p12 (client cert) ──────────────────────┐          │
└──────────────────────────────────────────────────────┼──────────┘
                                                       │
                                                       ▼
┌─────────────────────────────────────────────────────────────────┐
│                    Ubuntu 25.10 Server                           │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ┌───────────────────────────────────────────────────────────┐  │
│  │              WildFly 35.0.1.Final                         │  │
│  │     Port 8080 (HTTP) │ 8442 (HTTPS) │ 8443 (Admin)       │  │
│  │  ┌─────────────────────────────────────────────────────┐  │  │
│  │  │            EJBCA Community Edition v9.3             │  │  │
│  │  │                                                     │  │  │
│  │  │  ┌──────────────┐ ┌────────────────────────────┐   │  │  │
│  │  │  │ ManagementCA │ │ SassyCorp Root CA           │   │  │  │
│  │  │  │ (RSA 3072)   │ │ (ML-DSA-87, native token)  │   │  │  │
│  │  │  └──────────────┘ └──────────────┬─────────────┘   │  │  │
│  │  │                                  │                  │  │  │
│  │  │                   ┌──────────────▼─────────────┐   │  │  │
│  │  │                   │ SassyCorp Intermediate CA   │   │  │  │
│  │  │                   │ (ML-DSA-65, native token)  │   │  │  │
│  │  │                   └────────────────────────────┘   │  │  │
│  │  └─────────────────────────────────────────────────────┘  │  │
│  └───────────────────────────────────────────────────────────┘  │
│                              │                                  │
│                              ▼                                  │
│  ┌───────────────────────────────────────────────────────────┐  │
│  │              MariaDB (port 3306)                          │  │
│  │              Database: ejbca                              │  │
│  │  All CA keys, certificates, and crypto tokens stored here │  │
│  └───────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────┘
```

---

## What You've Accomplished

Let's take a step back and appreciate the full journey:

1. **Built the infrastructure** — You set up a MariaDB database, configured WildFly 35 with 3-port TLS separation using Elytron, and deployed EJBCA Community Edition from source.

2. **Created PQC CAs natively** — Using EJBCA's crypto token system, you created ML-DSA-87 and ML-DSA-65 certificate authorities directly within the platform — no external tools, no imports, just EJBCA's native PQC support.

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

- **Issue end-entity certificates** using the Intermediate CA through EJBCA's enrollment interface
- **Configure CRL distribution points** for your PQC CAs
- **Set up OCSP** for real-time certificate status checking
- **Create certificate profiles** tailored for different use cases (TLS server, code signing, email)
- **Explore the REST API** at `https://localhost:8443/ejbca/ejbca-rest-api/v1/`
- **Test hybrid certificates** combining PQC + traditional algorithms (EJBCA supports this)

---

## References

| Resource | URL |
|----------|-----|
| EJBCA CE Source | https://github.com/Keyfactor/ejbca-ce |
| EJBCA Documentation | https://docs.keyfactor.com/ejbca-software/latest/installation |
| EJBCA Security Guide | https://docs.keyfactor.com/ejbca/latest/ejbca-security |
| WildFly 35 Documentation | https://docs.wildfly.org/35/ |
| MariaDB Documentation | https://mariadb.com/kb/en/documentation/ |
| NIST FIPS 204 (ML-DSA) | https://csrc.nist.gov/pubs/fips/204/final |
| Keyfactor PQC Hybrid Tutorial | https://docs.keyfactor.com/ejbca/9.0/tutorial-create-pqc-hybrid-ca-chain |
| EJBCA REST API | https://docs.keyfactor.com/ejbca/latest/ejbca-rest-interface |

---

*This lab is part of the [Post-Quantum Cryptography Step-by-Step Lab](https://github.com/f5devcentral/openssl-pqc-stepbystep-lab) series. For educational and internal testing purposes. Production deployments should use HSMs, air-gapped Root CAs, and follow organizational security policies.*

---

*[← Previous: Module 08 - Finalize](08_ejbca_finalize.md) | [Back to Introduction →](00_ejbca_migration_intro.md)*
