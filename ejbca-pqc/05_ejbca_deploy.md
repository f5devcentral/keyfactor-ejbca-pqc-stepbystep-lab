# Module 05: Deploy EJBCA

## Time to Get the Code

This is where it all comes together. We're going to clone the EJBCA Community Edition source code from GitHub, configure all the properties files we discussed in Module 02, build the application, and deploy it to WildFly.

> **📋 Keyfactor Reference:** [Deploy EJBCA](https://docs.keyfactor.com/ejbca-software/latest/deploy-ejbca)

---

## Step 1: Clone EJBCA Community Edition

```bash
cd /opt
sudo git clone https://github.com/Keyfactor/ejbca-ce.git ejbca
```

This clones the EJBCA CE repository into `/opt/ejbca`.

<br>

## Step 2: Set Up Environment Variables

EJBCA's build system needs to know where WildFly and Java live:

```bash
export EJBCA_HOME=/opt/ejbca
export APPSRV_HOME=/opt/wildfly
export JAVA_HOME=/usr/lib/jvm/java-21-openjdk-$(dpkg --print-architecture)
```

Make these permanent:

```bash
sudo tee /etc/profile.d/ejbca.sh > /dev/null << 'EOF'
export EJBCA_HOME=/opt/ejbca
export APPSRV_HOME=/opt/wildfly
EOF
```

```bash
source /etc/profile.d/ejbca.sh
source /etc/profile.d/java.sh
```

<br>

## Step 3: Configure EJBCA Properties Files

Now we activate and edit each configuration file. This is where the Module 02 overview pays off.

### 3a: install.properties

```bash
cd /opt/ejbca/conf
sudo cp install.properties.sample install.properties
```

Edit the file:

```bash
sudo vi install.properties
```

Find and set these values (some may already be defaults, others need changing):

```properties
ca.name=ManagementCA
ca.dn=CN=ManagementCA,O=SassyCorp,C=US
ca.tokentype=soft
ca.tokenpassword=null
ca.keyspec=3072
ca.keytype=RSA
ca.signaturealgorithm=SHA256WithRSA
ca.validity=3650
ca.policy=null
ca.certificateprofile=ROOTCA
```

Save and exit vi.

> **💡 Reminder:** The Management CA uses RSA because it handles EJBCA's internal TLS and admin authentication. Our PQC CAs are created natively in EJBCA in Module 07.

### 3b: cesecore.properties

```bash
sudo cp cesecore.properties.sample cesecore.properties
```

```bash
sudo vi cesecore.properties
```

Set the password encryption key to a random string. Generate one (in a separate terminal):

```bash
openssl rand -base64 32
```

Copy that output and paste it as the value:

```properties
password.encryption.key=YOUR_RANDOM_STRING_HERE
ca.rngalgorithm=SHA1PRNG
ca.serialnumberoctetsize=20
```

Save and exit.

> **⚠️ Critical:** Write down or save this `password.encryption.key` somewhere safe. If you lose it and EJBCA has encrypted passwords stored in the database, they become unrecoverable. For this lab, saving it to a local text file is fine.

### 3c: ejbca.properties

```bash
sudo cp ejbca.properties.sample ejbca.properties
```

```bash
sudo vi ejbca.properties
```

Set:

```properties
appserver.home=/opt/wildfly
ejbca.productionmode=true
```

Save and exit.

### 3d: web.properties

```bash
sudo cp web.properties.sample web.properties
```

```bash
sudo vi web.properties
```

Set these values:

```properties
java.trustpassword=changeit
superadmin.cn=SuperAdmin
superadmin.dn=CN=SuperAdmin,O=SassyCorp,C=US
superadmin.password=ejbca
superadmin.batch=true
httpsserver.password=serverpwd
httpsserver.tokentype=P12
httpsserver.hostname=localhost
httpsserver.dn=CN=localhost,O=SassyCorp,C=US
httpserver.pubhttp=8080
httpserver.pubhttps=8442
httpserver.privhttps=8443
```

Save and exit.

### 3e: database.properties

```bash
sudo cp database.properties.sample database.properties
```

```bash
sudo vi database.properties
```

Set (several uncomments):

```properties
datasource.jndi-name=EjbcaDS
database.name=mysql
database.url=jdbc:mysql://127.0.0.1:3306/ejbca?permitMysqlScheme
database.driver=org.mariadb.jdbc.Driver
database.username=ejbca
database.password=ejbca
```

Save and exit.

> **💡 Note:** The `database.url`, `database.driver`, `database.username`, and `database.password` properties are used by EJBCA's CLI tools (like `ejbca-db-cli`). The main application connection to the database goes through the WildFly datasource we configured in Module 04. Both need to point to the same database.

<br>

## Step 4: Set Permissions

Make sure the build user can access the EJBCA directory:

```bash
sudo chown -R $(whoami):$(whoami) /opt/ejbca
```

<br>

## Step 5: Build and Deploy EJBCA

This is the moment of truth. The `ant deployear` command compiles EJBCA from source, packages it into an EAR file, and deploys it to WildFly.

```bash
cd /opt/ejbca
sudo ant -q clean deployear
```

**What happens during `ant deployear`:**

1. Cleans any previous build artifacts
2. Compiles all EJBCA Java source code
3. Packages the compiled code into `ejbca.ear`
4. Copies `ejbca.ear` to WildFly's deployment directory
5. WildFly hot-deploys the EAR automatically

This will take a few minutes. You'll see Ant output scrolling by. Watch for:

**Good signs:**
```
BUILD SUCCESSFUL
Total time: X minutes Y seconds
```

**Bad signs:**
```
BUILD FAILED
```

If the build fails, check:
- Is `JAVA_HOME` set? (`echo $JAVA_HOME`)
- Is `APPSRV_HOME` set? (`echo $APPSRV_HOME`)
- Are the properties files syntactically correct?
- Is there a Java version mismatch?

### Verify the Deployment

After the build succeeds, check WildFly's log:

```bash
sudo tail -30 /opt/wildfly/standalone/log/server.log
```

Look for:
```
WFLYSRV0010: Deployed "ejbca.ear"
```

And watch for any ERROR lines. Some warnings about missing optional configurations are normal — actual errors will be clearly marked.

<br>

## Step 6: Verify Database Indexes

EJBCA automatically creates all recommended database indexes during deployment. Let's confirm they're in place:

```bash
mysql -u ejbca -p -e "SELECT COUNT(DISTINCT INDEX_NAME) AS 'Indexes' FROM information_schema.STATISTICS WHERE TABLE_SCHEMA='ejbca' AND INDEX_NAME != 'PRIMARY';" ejbca
```

**Expected Output:**
```
+---------+
| Indexes |
+---------+
|      44 |
+---------+
```

> **💡 Note:** EJBCA's source tree includes an index creation script at `/opt/ejbca/doc/sql-scripts/create-index-ejbca.sql`, but you shouldn't need to run it — EJBCA creates these indexes automatically when the EAR deploys and initializes the database schema. The script exists as a safety net for edge cases where auto-creation is skipped.

If you got a different number just run the script and revalidate the Indexes:

```bash
mysql -u ejbca -p ejbca < /opt/ejbca/doc/sql-scripts/create-index-ejbca.sql
```

<br>

## Checkpoint

Verify the deployment:

```bash
echo "=== EJBCA EAR Deployed ==="
ls -la /opt/wildfly/standalone/deployments/ejbca.ear

echo ""
echo "=== WildFly Status ==="
sudo systemctl is-active wildfly-standalone

echo ""
echo "=== Deployment Status ==="
sudo -u wildfly /opt/wildfly/bin/jboss-cli.sh --connect '/deployment=ejbca.ear:read-attribute(name=status)' 2>/dev/null
```

The deployment status should show `"OK"`.

<br>

## What Just Happened?

Let's take a moment to appreciate what we just did:

1. **Cloned** A boatload of commits of open-source PKI software from GitHub
2. **Configured** five property files controlling every aspect of the installation
3. **Compiled** a Java enterprise application from source
4. **Deployed** it to a production-grade application server

EJBCA is now running inside WildFly, connected to MariaDB, and ready to be initialized as a Certificate Authority.

But it's not a CA yet. Right now it's just an application waiting to be told what to do. That happens in the next module when we run `ant runinstall`.

---

## Troubleshooting

### BUILD FAILED — javac error

Check your Java version matches:

```bash
java -version
javac -version
echo $JAVA_HOME
```

All should show Java 21.

### BUILD FAILED — APPSRV_HOME not set

```bash
export APPSRV_HOME=/opt/wildfly
```

### EJBCA won't deploy — missing datasource

Check the datasource exists in WildFly:

```bash
sudo -u wildfly /opt/wildfly/bin/jboss-cli.sh --connect '/subsystem=datasources/data-source=ejbcads:read-resource'
```

### EJBCA deploys but shows errors about database tables

This is normal on first deployment! EJBCA creates the tables automatically. Restart WildFly and the errors should resolve:

```bash
sudo systemctl restart wildfly-standalone
```

---

**Next Module:** [06 - Install EJBCA Software Stack](06_ejbca_install.md)

---

*[← Previous: Module 04 - WildFly](04_ejbca_wildfly.md) | [Next: Module 06 - Install EJBCA →](06_ejbca_install.md)*
