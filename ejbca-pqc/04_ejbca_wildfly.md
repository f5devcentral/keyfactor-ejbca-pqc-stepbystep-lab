# Module 04: WildFly 35 Application Server Setup

## What is WildFly and Why Do We Need It?

WildFly (formerly JBoss Application Server) is a [Jakarta EE](https://jakarta.ee/about/jakarta-ee/) application server. Think of it as the runtime environment that hosts and runs EJBCA — similar to how Apache or Nginx hosts web applications, but for Java enterprise applications.

EJBCA is a Java EE application packaged as an EAR (Enterprise Archive) file. WildFly provides the container services EJBCA needs: transaction management, database connection pooling, HTTPS/TLS termination, security, and remoting.

This module is the longest in the lab because WildFly requires careful configuration. Take your time — every JBoss CLI command matters.

> **📋 Keyfactor Reference:** [WildFly 35](https://docs.keyfactor.com/ejbca-software/latest/wildfly-35)

<br>

## Step 1: Download WildFly 35

We'll download the tar.gz package (Keyfactor also supports Galleon for minimal installs, but the tar.gz is more straightforward for learning).

```bash
wget https://github.com/wildfly/wildfly/releases/download/35.0.1.Final/wildfly-35.0.1.Final.tar.gz -O /tmp/wildfly-35.0.1.Final.tar.gz
```

Extract it to `/opt`:

```bash
sudo tar -xzf /tmp/wildfly-35.0.1.Final.tar.gz -C /opt/
```

Create a symlink so we can reference WildFly generically:

```bash
sudo ln -snf /opt/wildfly-35.0.1.Final /opt/wildfly
```

**Why the symlink?** Using `/opt/wildfly` as a symlink means that when WildFly is upgraded in the future, you just update the symlink to point to the new version — no need to reconfigure paths everywhere.

Verify:

```bash
ls -la /opt/wildfly
```

**Expected Output:**
```
lrwxrwxrwx 1 root root ... /opt/wildfly -> /opt/wildfly-35.0.1.Final
```

<br>

## Step 2: Remove RESTEasy-Crypto (Prevent Bouncy Castle Conflicts)

This is **critically important** for our PQC deployment. WildFly ships its own Bouncy Castle library inside the RESTEasy-Crypto module. If WildFly loads its Bouncy Castle instead of EJBCA's bundled version, you'll get ClassCastExceptions and cryptographic failures — especially with PQC algorithms.

```bash
sudo sed -i '/.*org.jboss.resteasy.resteasy-crypto.*/d' /opt/wildfly/modules/system/layers/base/org/jboss/as/jaxrs/main/module.xml
```

```bash
sudo rm -rf /opt/wildfly/modules/system/layers/base/org/jboss/resteasy/resteasy-crypto/
```

**What this does:**

1. Removes the reference to the resteasy-crypto module from the JAX-RS module descriptor
2. Deletes the resteasy-crypto module directory entirely (including its bundled Bouncy Castle)

> **⚠️ PQC Context:** This step is even more important for us than for a standard EJBCA install. EJBCA's bundled Bouncy Castle includes PQC algorithm support (ML-DSA, ML-KEM, etc.). If WildFly's older Bouncy Castle gets loaded instead, our ML-DSA certificates won't be recognized. The error you'd see: `BouncyCastle is not loaded by an EJBCA classloader, version conflict is likely`.

<br>

## Step 3: Create the WildFly Configuration

Replace the default `standalone.conf` with one optimized for EJBCA:

```bash
sudo tee /opt/wildfly/bin/standalone.conf > /dev/null << 'CONFEOF'
if [ "x$JBOSS_MODULES_SYSTEM_PKGS" = "x" ]; then
     JBOSS_MODULES_SYSTEM_PKGS="org.jboss.byteman"
fi

if [ "x$JAVA_OPTS" = "x" ]; then
     JAVA_OPTS="-Xms2048m -Xmx2048m"
     JAVA_OPTS="$JAVA_OPTS -Dhttps.protocols=TLSv1.2,TLSv1.3"
     JAVA_OPTS="$JAVA_OPTS -Djdk.tls.client.protocols=TLSv1.2,TLSv1.3"
     JAVA_OPTS="$JAVA_OPTS -Djava.net.preferIPv4Stack=true"
     JAVA_OPTS="$JAVA_OPTS -Djboss.modules.system.pkgs=$JBOSS_MODULES_SYSTEM_PKGS"
     JAVA_OPTS="$JAVA_OPTS -Djava.awt.headless=true"
     JAVA_OPTS="$JAVA_OPTS -XX:+HeapDumpOnOutOfMemoryError"
     JAVA_OPTS="$JAVA_OPTS -Djdk.tls.ephemeralDHKeySize=2048"
else
     echo "JAVA_OPTS already set in environment; overriding default settings with values: $JAVA_OPTS"
fi
CONFEOF
```

**Key settings explained:**

| Setting | Value | Purpose |
|---------|-------|---------|
| `-Xms2048m -Xmx2048m` | 2 GB heap | Minimum for EJBCA (default 512 MB is insufficient) |
| `TLSv1.2,TLSv1.3` | TLS versions | Only allow modern TLS protocols |
| `preferIPv4Stack` | true | Avoid IPv6 issues in lab environments |
| `headless` | true | No GUI needed on a server |
| `HeapDumpOnOutOfMemoryError` | enabled | Debugging aid if memory runs out |
| `ephemeralDHKeySize=2048` | 2048-bit DH | Strong ephemeral Diffie-Hellman parameters |

### Set the Transaction Node ID

Each WildFly instance needs a unique transaction node ID:

```bash
echo "JAVA_OPTS=\"\$JAVA_OPTS -Djboss.tx.node.id=$(od -A n -t d -N 1 /dev/urandom | tr -d ' ')\"" | sudo tee -a /opt/wildfly/bin/standalone.conf
```

### Add PKCS#11 Export for EJBCA Community Edition

EJBCA CE needs this additional Java option to access PKCS#11 token internals:

```bash
echo 'JAVA_OPTS="$JAVA_OPTS --add-exports=jdk.crypto.cryptoki/sun.security.pkcs11.wrapper=ALL-UNNAMED"' | sudo tee -a /opt/wildfly/bin/standalone.conf
```

<br>

## Step 4: Fix the WildFly Launch Script

The default `launch.sh` script that WildFly's systemd service uses has a problem: it uses `#!/bin/sh` as the shebang but contains bash-specific syntax. On Ubuntu, `/bin/sh` is `dash`, not `bash`, so the script silently fails with `[: unexpected operator` on line 2 — and `systemctl start wildfly` just... does nothing. No error, no log, nothing.

Fix the shebang:

```bash
sudo sed -i '1s|#!/bin/sh|#!/bin/bash|' /opt/wildfly/bin/systemd/launch.sh
```

Verify:

```bash
head -1 /opt/wildfly/bin/systemd/launch.sh
```

**Expected Output:** `#!/bin/bash`

<br>

## Step 5: Configure WildFly as a Systemd Service

Create the `wildfly` system user:

```bash
sudo useradd -r -s /bin/false wildfly
```

Copy the systemd service files that come with WildFly:

```bash
sudo cp /opt/wildfly/bin/systemd/wildfly-standalone.service /etc/systemd/system/
```

Create the sysconfig directory if it doesn't exist (needed on Debian/Ubuntu):

```bash
sudo mkdir -p /etc/sysconfig
```

```bash
sudo cp /opt/wildfly/bin/systemd/wildfly-standalone.conf /etc/sysconfig/
```

The default sysconfig file has all variables commented out. WildFly won't start without them. Edit the file:

```bash
sudo vi /etc/sysconfig/wildfly-standalone.conf
```

Uncomment and set these values:

```bash
WILDFLY_SH="/opt/wildfly/bin/standalone.sh"
WILDFLY_SERVER_CONFIG=standalone.xml
WILDFLY_BIND=0.0.0.0
WILDFLY_CONSOLE_LOG=/opt/wildfly/standalone/log/service_console.log
```

Additionally add the JAVA_HOME path you defined prior based on arm or amd. I'm using arm.

```bash
#JAVA_HOME=/usr/lib/jvm/jre
# Use the following for location of java in the SDK
#JAVA_HOME=/usr/lib/jvm/java
JAVA_HOME=/usr/lib/jvm/java-21-openjdk-arm64
```

Save and exit. Without these, `launch.sh` has no script to call and exits silently — WildFly will show `active` for a split second then go `dead` with no error.

Set ownership:

```bash
sudo chown -R wildfly:wildfly /opt/wildfly-35.0.1.Final/
```

Reload systemd:

```bash
sudo systemctl daemon-reload
```

<br>

## Step 6: Start WildFly

```bash
sudo systemctl start wildfly-standalone
```

Check it started successfully:

```bash
sudo systemctl status wildfly-standalone
```

**Expected:** `active (running)`

If it fails, check the log:

```bash
sudo journalctl -u wildfly-standalone --no-pager -n 50
```

Also check WildFly's own log:

```bash
sudo tail -50 /opt/wildfly/standalone/log/server.log
```

Enable auto-start on boot:

```bash
sudo systemctl enable wildfly-standalone
```

<br>

## Step 7: Create the Elytron Credential Store

The credential store encrypts sensitive passwords (like the database password) so they're not stored in plain text in WildFly's configuration.

### Create the Master Password Script

```bash
sudo sh -c 'echo "#!/bin/sh" > /opt/wildfly/bin/wildfly_pass'
sudo sh -c "echo \"echo '$(openssl rand -base64 24)'\" >> /opt/wildfly/bin/wildfly_pass"
sudo chown wildfly:wildfly /opt/wildfly/bin/wildfly_pass
sudo chmod 700 /opt/wildfly/bin/wildfly_pass
```

**What this does:** Creates a script that outputs a random master password. Only the `wildfly` user can execute it. WildFly calls this script at startup to unlock the credential store.

### Create the Credential Store

```bash
sudo mkdir -p /opt/wildfly/standalone/configuration/keystore
sudo chown wildfly:wildfly /opt/wildfly/standalone/configuration/keystore
```

```bash
sudo -u wildfly /opt/wildfly/bin/jboss-cli.sh --connect '/subsystem=elytron/credential-store=defaultCS:add(path=keystore/credentials, relative-to=jboss.server.config.dir, credential-reference={clear-text="{EXT}/opt/wildfly/bin/wildfly_pass", type="COMMAND"}, create=true)'
```

**Expected Output:** `{"outcome" => "success"}`

<br>

## Step 8: Add the MariaDB JDBC Driver

WildFly needs a JDBC driver to talk to MariaDB. We'll hot-deploy it:

```bash
sudo wget https://dlm.mariadb.com/3978472/Connectors/java/connector-java-3.5.1/mariadb-java-client-3.5.1.jar -O /opt/wildfly/standalone/deployments/mariadb-java-client.jar
```

```bash
sudo chown wildfly:wildfly /opt/wildfly/standalone/deployments/mariadb-java-client.jar
```

Wait a few seconds for WildFly to pick up the driver, then verify:

```bash
sudo tail -5 /opt/wildfly/standalone/log/server.log | grep -i mariadb
```

You should see a log line about the MariaDB driver being deployed.

<br>

## Step 9: Add the EJBCA Datasource

This connects EJBCA to the MariaDB database we created in Module 03.

First, store the database password in the credential store:

```bash
sudo -u wildfly /opt/wildfly/bin/jboss-cli.sh --connect '/subsystem=elytron/credential-store=defaultCS:add-alias(alias=dbPassword, secret-value="ejbca")'
```

Now create the datasource:

```bash
sudo -u wildfly /opt/wildfly/bin/jboss-cli.sh --connect 'data-source add --name=ejbcads --connection-url="jdbc:mysql://127.0.0.1:3306/ejbca?permitMysqlScheme" --jndi-name="java:/EjbcaDS" --use-ccm=true --driver-name="mariadb-java-client.jar" --driver-class="org.mariadb.jdbc.Driver" --user-name="ejbca" --credential-reference={store=defaultCS, alias=dbPassword} --validate-on-match=true --background-validation=false --prepared-statements-cache-size=50 --share-prepared-statements=true --min-pool-size=5 --max-pool-size=150 --pool-prefill=true --transaction-isolation=TRANSACTION_READ_COMMITTED --check-valid-connection-sql="select 1;"'
```

**What this command does:**
- Creates a datasource named `ejbcads`
- Connects to MariaDB at `127.0.0.1:3306/ejbca` with `?permitMysqlScheme` (required for MariaDB connector 3.x+)
- Sets the JNDI name to `java:/EjbcaDS` (must match `datasource.jndi-name` in EJBCA's `database.properties`)
- Uses the encrypted password from the credential store
- Configures connection pooling (5–150 connections)
- Sets transaction isolation to READ_COMMITTED

Reload WildFly to apply:

```bash
sudo -u wildfly /opt/wildfly/bin/jboss-cli.sh --connect ':reload'
```

**⚠️ Wait for the reload to complete before continuing.** Check with:

```bash
sudo -u wildfly /opt/wildfly/bin/jboss-cli.sh --connect ':read-attribute(name=server-state)'
```

**Expected Output:** `"running"`

<br>

## Step 10: Configure WildFly Remoting

EJBCA's CLI tools use JBoss Remoting to communicate with the server. Configure a dedicated port:

```bash
sudo -u wildfly /opt/wildfly/bin/jboss-cli.sh --connect '/subsystem=remoting/http-connector=http-remoting-connector:write-attribute(name=connector-ref,value=remoting)'
```

```bash
sudo -u wildfly /opt/wildfly/bin/jboss-cli.sh --connect '/socket-binding-group=standard-sockets/socket-binding=remoting:add(port=4447,interface=management)'
```

```bash
sudo -u wildfly /opt/wildfly/bin/jboss-cli.sh --connect '/subsystem=undertow/server=default-server/http-listener=remoting:add(socket-binding=remoting,enable-http2=true)'
```

Reload:

```bash
sudo -u wildfly /opt/wildfly/bin/jboss-cli.sh --connect ':reload'
```

Wait for `"running"` state again:

```bash
sudo -u wildfly /opt/wildfly/bin/jboss-cli.sh --connect ':read-attribute(name=server-state)'
```

<br>

## Step 11: Configure Logging

Set up EJBCA-specific logging:

```bash
sudo -u wildfly /opt/wildfly/bin/jboss-cli.sh --connect '/subsystem=logging/logger=org.ejbca:add(level=INFO)'
sudo -u wildfly /opt/wildfly/bin/jboss-cli.sh --connect '/subsystem=logging/logger=org.cesecore:add(level=INFO)'
sudo -u wildfly /opt/wildfly/bin/jboss-cli.sh --connect '/subsystem=logging/logger=com.keyfactor:add(level=INFO)'
```

Reduce noise from other components:

```bash
sudo -u wildfly /opt/wildfly/bin/jboss-cli.sh --connect '/subsystem=logging/logger=org.jboss.as.config:write-attribute(name=level, value=WARN)'
```

<br>

## Step 12: Configure TLS for EJBCA (3-Port Separation)

This is where we configure WildFly's Elytron security subsystem for EJBCA's TLS requirements. Without this configuration, `ant deploy-keystore` creates the keystores but WildFly has no TLS listeners to use them.

EJBCA uses a **3-port separation** model per [Keyfactor's documentation](https://docs.keyfactor.com/ejbca/9.2/wildfly-35):

| Port | Protocol | Client Cert | Purpose |
|------|----------|-------------|---------|
| **8080** | HTTP | None | Public pages, health checks |
| **8442** | HTTPS | Server only | Public HTTPS access |
| **8443** | HTTPS | Mutual TLS | Admin web (client certificate required) |

### 12a: Remove Existing Default HTTP/HTTPS Listeners

WildFly ships with default HTTP listeners we need to replace. Remove them:

```bash
sudo -u wildfly /opt/wildfly/bin/jboss-cli.sh --connect '/subsystem=undertow/server=default-server/http-listener=default:remove()'
```

```bash
sudo -u wildfly /opt/wildfly/bin/jboss-cli.sh --connect '/subsystem=undertow/server=default-server/https-listener=https:remove()'
```

```bash
sudo -u wildfly /opt/wildfly/bin/jboss-cli.sh --connect '/socket-binding-group=standard-sockets/socket-binding=http:remove()'
```

```bash
sudo -u wildfly /opt/wildfly/bin/jboss-cli.sh --connect '/socket-binding-group=standard-sockets/socket-binding=https:remove()'
```

Reload and wait:

```bash
sudo -u wildfly /opt/wildfly/bin/jboss-cli.sh --connect ':reload'
```

```bash
sleep 10 && sudo -u wildfly /opt/wildfly/bin/jboss-cli.sh --connect ':read-attribute(name=server-state)'
```

> **⚠️ Note:** Some of these `remove()` commands may return errors if the resources don't exist (e.g., if you used Galleon or already removed them). That's fine — it's just making sure we start clean.

### 12b: Add Network Interfaces and Socket Bindings

Create dedicated interfaces for each port:

```bash
sudo -u wildfly /opt/wildfly/bin/jboss-cli.sh --connect '/interface=http:add(inet-address="0.0.0.0")'
sudo -u wildfly /opt/wildfly/bin/jboss-cli.sh --connect '/interface=httpspub:add(inet-address="0.0.0.0")'
sudo -u wildfly /opt/wildfly/bin/jboss-cli.sh --connect '/interface=httpspriv:add(inet-address="0.0.0.0")'
```

Bind ports to those interfaces:

```bash
sudo -u wildfly /opt/wildfly/bin/jboss-cli.sh --connect '/socket-binding-group=standard-sockets/socket-binding=http:add(port="8080",interface="http")'
sudo -u wildfly /opt/wildfly/bin/jboss-cli.sh --connect '/socket-binding-group=standard-sockets/socket-binding=httpspub:add(port="8442",interface="httpspub")'
sudo -u wildfly /opt/wildfly/bin/jboss-cli.sh --connect '/socket-binding-group=standard-sockets/socket-binding=httpspriv:add(port="8443",interface="httpspriv")'
```

> **💡 What `0.0.0.0` means:** Binding to all interfaces. In production, you'd bind to specific IPs. For our lab this is fine.

### 12c: Configure TLS Keystores and SSL Contexts

Store the keystore and truststore passwords in the Elytron credential store:

```bash
sudo -u wildfly /opt/wildfly/bin/jboss-cli.sh --connect '/subsystem=elytron/credential-store=defaultCS:add-alias(alias=httpsKeystorePassword, secret-value="serverpwd")'
```

```bash
sudo -u wildfly /opt/wildfly/bin/jboss-cli.sh --connect '/subsystem=elytron/credential-store=defaultCS:add-alias(alias=httpsTruststorePassword, secret-value="changeit")'
```

Create the Elytron key stores. These point to PKCS#12 files that **don't exist yet** — `ant deploy-keystore` will create them in Module 06:

```bash
sudo -u wildfly /opt/wildfly/bin/jboss-cli.sh --connect '/subsystem=elytron/key-store=httpsKS:add(path="keystore/keystore.p12",relative-to=jboss.server.config.dir,credential-reference={store=defaultCS, alias=httpsKeystorePassword},type=PKCS12)'
```

```bash
sudo -u wildfly /opt/wildfly/bin/jboss-cli.sh --connect '/subsystem=elytron/key-store=httpsTS:add(path="keystore/truststore.p12",relative-to=jboss.server.config.dir,credential-reference={store=defaultCS, alias=httpsTruststorePassword},type=PKCS12)'
```

> **⚠️ Critical: PKCS12, not JKS.** EJBCA's `ant deploy-keystore` creates `keystore.p12` and `truststore.p12` (PKCS#12 format), not `tomcat.jks`/`truststore.jks` (JKS format). The type and file paths here **must** match what EJBCA generates. Getting this wrong causes TLS handshake failures. Also ensure `httpsserver.tokentype=P12` is set in `web.properties` (Module 05).

Create the key manager and trust manager:

```bash
sudo -u wildfly /opt/wildfly/bin/jboss-cli.sh --connect '/subsystem=elytron/key-manager=httpsKM:add(key-store=httpsKS,algorithm="SunX509",credential-reference={store=defaultCS, alias=httpsKeystorePassword})'
```

```bash
sudo -u wildfly /opt/wildfly/bin/jboss-cli.sh --connect '/subsystem=elytron/trust-manager=httpsTM:add(key-store=httpsTS)'
```

Create two SSL contexts — one for public HTTPS (server cert only) and one for admin HTTPS (mutual TLS):

```bash
sudo -u wildfly /opt/wildfly/bin/jboss-cli.sh --connect '/subsystem=elytron/server-ssl-context=httpspub:add(key-manager=httpsKM,protocols=["TLSv1.3","TLSv1.2"],use-cipher-suites-order=false,cipher-suite-filter="TLS_DHE_RSA_WITH_AES_128_GCM_SHA256,TLS_ECDHE_RSA_WITH_AES_128_GCM_SHA256,TLS_ECDHE_ECDSA_WITH_AES_128_GCM_SHA256,TLS_ECDHE_RSA_WITH_CHACHA20_POLY1305_SHA256,TLS_ECDHE_ECDSA_WITH_CHACHA20_POLY1305_SHA256",cipher-suite-names="TLS_AES_256_GCM_SHA384:TLS_AES_128_GCM_SHA256:TLS_CHACHA20_POLY1305_SHA256")'
```

```bash
sudo -u wildfly /opt/wildfly/bin/jboss-cli.sh --connect '/subsystem=elytron/server-ssl-context=httpspriv:add(key-manager=httpsKM,protocols=["TLSv1.3","TLSv1.2"],use-cipher-suites-order=false,cipher-suite-filter="TLS_DHE_RSA_WITH_AES_128_GCM_SHA256,TLS_ECDHE_RSA_WITH_AES_128_GCM_SHA256,TLS_ECDHE_ECDSA_WITH_AES_128_GCM_SHA256,TLS_ECDHE_RSA_WITH_CHACHA20_POLY1305_SHA256,TLS_ECDHE_ECDSA_WITH_CHACHA20_POLY1305_SHA256",cipher-suite-names="TLS_AES_256_GCM_SHA384:TLS_AES_128_GCM_SHA256:TLS_CHACHA20_POLY1305_SHA256",trust-manager=httpsTM,need-client-auth=true)'
```

**SSL Context explained:**

| Context | Port | Client Cert | Purpose |
|---------|------|-------------|---------|
| `httpspub` | 8442 | Not required | Public HTTPS — enrollment pages, health checks over TLS |
| `httpspriv` | 8443 | Required (mutual TLS) | Admin UI — browser must present client cert, server validates against trust store |

> **💡 The Key Difference:** `httpspub` has only a key-manager (server presents its cert). `httpspriv` has both a key-manager AND a trust-manager with `need-client-auth=true` — this is what forces browsers to present a client certificate on port 8443. This matches [Keyfactor's 3-port separation](https://docs.keyfactor.com/ejbca/9.2/wildfly-35) specification.

### 12d: Add Undertow HTTP/HTTPS Listeners

Now wire the listeners to their socket bindings and SSL contexts:

```bash
sudo -u wildfly /opt/wildfly/bin/jboss-cli.sh --connect '/subsystem=undertow/server=default-server/http-listener=http:add(socket-binding="http", redirect-socket="httpspriv")'
```

```bash
sudo -u wildfly /opt/wildfly/bin/jboss-cli.sh --connect '/subsystem=undertow/server=default-server/https-listener=httpspub:add(socket-binding="httpspub", ssl-context="httpspub", max-parameters=2048)'
```

```bash
sudo -u wildfly /opt/wildfly/bin/jboss-cli.sh --connect '/subsystem=undertow/server=default-server/https-listener=httpspriv:add(socket-binding="httpspriv", ssl-context="httpspriv", max-parameters=2048)'
```

### 12e: Add HTTP Protocol Behavior System Properties

These system properties configure encoding and protocol behavior for EJBCA's web services:

```bash
sudo -u wildfly /opt/wildfly/bin/jboss-cli.sh --connect '/system-property=org.apache.catalina.connector.URI_ENCODING:add(value="UTF-8")'
```

```bash
sudo -u wildfly /opt/wildfly/bin/jboss-cli.sh --connect '/system-property=org.apache.catalina.connector.USE_BODY_ENCODING_FOR_QUERY_STRING:add(value=true)'
```

```bash
sudo -u wildfly /opt/wildfly/bin/jboss-cli.sh --connect '/system-property=org.apache.tomcat.util.buf.UDecoder.ALLOW_ENCODED_SLASH:add(value=true)'
```

```bash
sudo -u wildfly /opt/wildfly/bin/jboss-cli.sh --connect '/system-property=org.apache.tomcat.util.http.Parameters.MAX_COUNT:add(value=2048)'
```

```bash
sudo -u wildfly /opt/wildfly/bin/jboss-cli.sh --connect '/system-property=org.apache.catalina.connector.CoyoteAdapter.ALLOW_BACKSLASH:add(value=true)'
```

```bash
sudo -u wildfly /opt/wildfly/bin/jboss-cli.sh --connect '/subsystem=webservices:write-attribute(name=wsdl-host, value=jbossws.undefined.host)'
```

```bash
sudo -u wildfly /opt/wildfly/bin/jboss-cli.sh --connect '/subsystem=webservices:write-attribute(name=modify-wsdl-address, value=true)'
```

### 12f: Reload and Verify

```bash
sudo -u wildfly /opt/wildfly/bin/jboss-cli.sh --connect ':reload'
```

Wait for reload, then verify all three ports are listening:

```bash
sleep 15 && sudo ss -tlnp | grep -E '8080|8442|8443'
```

**Expected Output:** You should see all three ports listening:

```
LISTEN  0  128  0.0.0.0:8080   0.0.0.0:*
LISTEN  0  128  0.0.0.0:8442   0.0.0.0:*
LISTEN  0  128  0.0.0.0:8443   0.0.0.0:*
```

> **🔥 If port 8442 is missing** or 8443 isn't showing, something went wrong in the commands above. Check WildFly's log: `sudo tail -50 /opt/wildfly/standalone/log/server.log` and look for Elytron or Undertow errors.

> **💡 Note:** WildFly will log warnings about the keystore files not existing yet. This is expected — we're configuring *where* WildFly should look for them. The actual files get created by `ant deploy-keystore` in Module 06.

<br>

## Checkpoint

Let's verify the WildFly setup (this you can copy/paste):

```bash
echo "=== WildFly Service ==="
sudo systemctl is-active wildfly-standalone

echo "=== WildFly Version ==="
sudo -u wildfly /opt/wildfly/bin/jboss-cli.sh --connect ':read-attribute(name=product-version)' 2>/dev/null

echo "=== Datasource Test ==="
sudo -u wildfly /opt/wildfly/bin/jboss-cli.sh --connect '/subsystem=datasources/data-source=ejbcads:test-connection-in-pool' 2>/dev/null

echo "=== Credential Store ==="
sudo -u wildfly /opt/wildfly/bin/jboss-cli.sh --connect '/subsystem=elytron/credential-store=defaultCS:read-aliases' 2>/dev/null
```

The datasource connection test is the most critical — it proves WildFly can talk to MariaDB through the JDBC driver. You should see `"outcome" => "success"`.

> **💡 Note:** The 3-port verification (`ss -tlnp` for ports 8080/8442/8443) only applies after you've completed Step 12, which runs after Module 06.

<br>

## WildFly Architecture Summary

Here's what we built:

```
┌──────────────────────────────────────────┐
│           WildFly 35.0.1.Final           │
│      /opt/wildfly → /opt/wildfly-35...   │
│      Service: wildfly-standalone         │
│      User: wildfly                       │
├──────────────────────────────────────────┤
│  Ports (after Step 12):                  │
│    8080  — HTTP (public)                 │
│    8442  — HTTPS (public, no client cert)│
│    8443  — HTTPS (admin, mTLS required)  │
│    4447  — JBoss Remoting (CLI)          │
├──────────────────────────────────────────┤
│  TLS (Elytron):                          │
│    httpsKS → keystore/keystore.p12       │
│    httpsTS → keystore/truststore.p12     │
│    httpspub  → port 8442, server only    │
│    httpspriv → port 8443, mutual TLS     │
├──────────────────────────────────────────┤
│  Datasource: java:/EjbcaDS               │
│    → MariaDB 127.0.0.1:3306/ejbca        │
│    → JDBC: mariadb-java-client.jar       │
│    → Password: Elytron credential store  │
├──────────────────────────────────────────┤
│  Java: OpenJDK 21 (2 GB heap)            │
│  Bouncy Castle: RESTEasy-Crypto REMOVED  │
└──────────────────────────────────────────┘
```

---

## Troubleshooting

### WildFly won't start — silently fails

First check if it's the launch.sh shebang issue:

```bash
sudo journalctl -u wildfly-standalone --no-pager -n 20 | grep -i "unexpected operator"
```

If you see `[: unexpected operator`, the launch.sh fix from Step 4 wasn't applied. Fix it:

```bash
sudo sed -i '1s|#!/bin/sh|#!/bin/bash|' /opt/wildfly/bin/systemd/launch.sh
sudo systemctl restart wildfly-standalone
```

### WildFly won't start — port in use

Check if port 8080 is already in use:

```bash
sudo ss -tlnp | grep 8080
```

If another service is using it, stop that service or change the port.

### Datasource connection test fails

Verify MariaDB is running:

```bash
sudo systemctl status mariadb
```

Verify the database and user exist (test the TCP connection WildFly uses):

```bash
mariadb -u ejbca -p -h 127.0.0.1 -e "SELECT 1;" ejbca
```

### "BouncyCastle is not loaded by an EJBCA classloader"

You missed Step 2 (removing RESTEasy-Crypto). Go back and run those commands, then restart WildFly:

```bash
sudo systemctl restart wildfly-standalone
```

### JBoss CLI can't connect

Make sure WildFly is fully started. The CLI connects via the management interface which takes a moment to initialize:

```bash
sleep 10 && sudo -u wildfly /opt/wildfly/bin/jboss-cli.sh --connect ':read-attribute(name=server-state)'
```

### Port 8442 not listening after Step 12

Check that the httpspub listener was created correctly:

```bash
sudo -u wildfly /opt/wildfly/bin/jboss-cli.sh --connect '/subsystem=undertow/server=default-server/https-listener=httpspub:read-resource'
```

If it returns "not found," re-run the Step 12 commands starting from 12c (ssl-contexts) and 12d (listeners).

### Port 8443 not requiring client certificate

Verify the httpspriv SSL context has `need-client-auth=true`:

```bash
sudo -u wildfly /opt/wildfly/bin/jboss-cli.sh --connect '/subsystem=elytron/server-ssl-context=httpspriv:read-attribute(name=need-client-auth)'
```

Should return `true`. If not:

```bash
sudo -u wildfly /opt/wildfly/bin/jboss-cli.sh --connect '/subsystem=elytron/server-ssl-context=httpspriv:write-attribute(name=need-client-auth,value=true)'
sudo -u wildfly /opt/wildfly/bin/jboss-cli.sh --connect ':reload'
```

### Out of memory errors

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

<br>

**Next Module:** [05 - Deploy EJBCA](05_ejbca_deploy.md)

---

*[← Previous: Module 03 - Database](03_ejbca_database.md) | [Next: Module 05 - Deploy EJBCA →](05_ejbca_deploy.md)*
