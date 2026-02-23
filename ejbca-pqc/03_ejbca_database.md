# Module 03: Database Setup — MariaDB

## Why a Database?

EJBCA stores everything in a relational database: CA configurations, certificate data, audit logs, user accounts, certificate profiles, CRLs, and more. While EJBCA bundles an embedded database for quick demos, any serious deployment uses an external database.

We're using MariaDB because it's recommended by Keyfactor, it's free, it's well-supported on Ubuntu, and it handles EJBCA's workload beautifully.

> **📋 Keyfactor Reference:** [Create the Database](https://docs.keyfactor.com/ejbca-software/latest/create-the-database)

---

## Step 1: Install MariaDB

```bash
sudo apt install -y mariadb-server mariadb-client
```

**What this installs:**
- `mariadb-server` — The database server itself
- `mariadb-client` — The `mysql` command-line client (yes, it's called `mysql` even for MariaDB)

### Verify the Installation

```bash
sudo systemctl status mariadb
```

**Expected Output:** You should see `active (running)`. If it's not running:

```bash
sudo systemctl start mariadb
sudo systemctl enable mariadb
```

The `enable` command ensures MariaDB starts automatically on boot.

### Check the Version

```bash
mariadb --version
```

**Expected Output:**

```text
mariadb from 11.8.3-MariaDB, client 15.2 for debian-linux-gnu ...
```

---

## Step 2: Secure the MariaDB Installation

MariaDB ships with some insecure defaults. Let's lock it down:

```bash
sudo mariadb-secure-installation
```

This interactive script will ask you several questions. Here's how to answer them for our lab:

| Question | Answer | Why |
|----------|--------|-----|
| Enter current password for root | (press Enter — no password set yet) | Fresh install has no root password |
| Switch to unix_socket authentication? | `n` | We'll use password auth |
| Set root password? | `Y` | Set a root password you'll remember |
| Remove anonymous users? | `Y` | Security best practice |
| Disallow root login remotely? | `Y` | Root should only connect locally |
| Remove test database? | `Y` | Don't need it |
| Reload privilege tables? | `Y` | Apply changes immediately |

> **💡 Remember Your Root Password:** You'll need the MariaDB root password in the next step to create the EJBCA database and user.

---

## Step 3: Create the EJBCA Database

Log into MariaDB as root:

```bash
sudo mariadb -u root -p
```

Enter the root password you just set.

Now create the EJBCA database with the correct character set. This is critical — EJBCA requires UTF-8:

```sql
CREATE DATABASE ejbca CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;
```

**What this does:**
- Creates a database named `ejbca`
- Sets character encoding to `utf8mb4` (full UTF-8 with 4-byte character support)
- Sets the collation to `utf8mb4_unicode_ci` (case-insensitive Unicode sorting)

> **⚠️ Character Set Note:** We use `utf8mb4` (full 4-byte UTF-8) which is the modern best practice. In rare cases, you may hit a `Specified key was too long; max key length is 767 bytes` error during EJBCA deployment. If that happens, see the Troubleshooting section at the end of this module for the `utf8` fallback.

---

## Step 4: Create the EJBCA Database User

Still in the MariaDB shell, create a dedicated user for EJBCA:

```sql
CREATE USER 'ejbca'@'localhost' IDENTIFIED BY 'ejbca';
CREATE USER 'ejbca'@'127.0.0.1' IDENTIFIED BY 'ejbca';
```

Grant the user full privileges on the EJBCA database:

```sql
GRANT ALL PRIVILEGES ON ejbca.* TO 'ejbca'@'localhost';
GRANT ALL PRIVILEGES ON ejbca.* TO 'ejbca'@'127.0.0.1';
```

Flush privileges to apply the changes:

```sql
FLUSH PRIVILEGES;
```

> **⚠️ Lab vs. Production:** We're using `ejbca` as both the username and password for simplicity in this learning lab. In production, use a strong, randomly generated password and store it securely. You should also restrict table-level privileges — see Keyfactor's [Database Privileges](https://docs.keyfactor.com/ejbca/latest/ejbca-security) documentation.

---

## Step 5: Verify the Database and User

Still in the MariaDB shell, verify everything was created correctly:

```sql
SHOW DATABASES;
```

**Expected Output:** You should see `ejbca` in the list:
```
+--------------------+
| Database           |
+--------------------+
| ejbca              |
| information_schema |
| mysql              |
| performance_schema |
+--------------------+
```

Check the user exists:

```sql
SELECT user, host FROM mysql.user WHERE user = 'ejbca';
```

**Expected Output:**
```
+-------+-----------+
| User  | Host      |
+-------+-----------+
| ejbca | 127.0.0.1 |
| ejbca | localhost |
+-------+-----------+
```

Exit the MariaDB shell:

```sql
EXIT;
```

---

## Step 6: Test the EJBCA User Connection

We need to verify the `ejbca` user can connect via **both** Unix socket and TCP. This matters because WildFly's JDBC URL uses `jdbc:mysql://127.0.0.1:3306/ejbca` which connects over TCP, not the Unix socket. MariaDB treats `localhost` (socket) and `127.0.0.1` (TCP) as different hosts — that's why we created the user for both in Step 4.

### Test Unix Socket Connection

```bash
mariadb -u ejbca -p -e "STATUS;" ejbca
```

Enter the password `ejbca` when prompted. In the output, look for the **Connection** line:

```
Connection:		Localhost via UNIX socket
```

This confirms the `ejbca@localhost` user works over the Unix socket.

### Test TCP Connection

```bash
mariadb -u ejbca -p -h 127.0.0.1 -e "STATUS;" ejbca
```

Enter the password `ejbca` when prompted. This time the **Connection** line should show:

```
Connection:		127.0.0.1 via TCP/IP
```

This confirms the `ejbca@127.0.0.1` user works over TCP — which is how WildFly will connect.

> **💡 Why both?** If only the `localhost` user exists, the EJBCA CLI tools (which use the socket) will work fine, but WildFly's JDBC datasource (which uses TCP to `127.0.0.1`) will get `Access denied`. Testing both now saves a frustrating debugging session in Module 04.

---

## Step 7: Verify Binary Log Format

EJBCA requires `binlog_format=ROW`. Let's check the current setting:

```bash
mariadb -u ejbca -p -e "SHOW VARIABLES LIKE 'binlog_format';"
```

**Expected Output:**
```
+---------------+-------+
| Variable_name | Value |
+---------------+-------+
| binlog_format | ROW   |
+---------------+-------+
```

If it shows `STATEMENT` instead of `ROW`, we should fix it (mine said `mixed` and while fine, we'll still fix):

```bash
sudo vi /etc/mysql/mariadb.conf.d/50-server.cnf
```

Add or modify under the `[mysqld]` section:

```ini
binlog_format=ROW
```

Then restart MariaDB:

```bash
sudo systemctl restart mariadb
```

Rerun the `SHOW VARIABLES` statement from before to validate the change.

<br>

## What About Database Indexes?

EJBCA includes SQL scripts for creating recommended database indexes. These improve query performance as your certificate database grows. The scripts are located in the EJBCA source tree at:

```text
/opt/ejbca/doc/sql-scripts/create-index-ejbca.sql
```

We'll apply these after EJBCA is deployed in Module 05. For now, the database structure is ready.

---

## Database Architecture Summary

Here's what we just built:

```
┌─────────────────────────────────┐
│         MariaDB Server          │
│    (systemd: mariadb.service)   │
├─────────────────────────────────┤
│  Database: ejbca                │
│  Character Set: utf8mb4         │
│  Collation: utf8mb4_unicode_ci  │
├─────────────────────────────────┤
│  User: ejbca@localhost (socket) │
│  User: ejbca@127.0.0.1 (TCP)    │
│  Privileges: ALL on ejbca.*     │
├─────────────────────────────────┤
│  Port: 3306 (localhost only)    │
│  Binary Log: ROW format         │
└─────────────────────────────────┘
```

---

## Troubleshooting

### Can't connect to MariaDB

```bash
sudo systemctl status mariadb
```

If it's not running, check the logs:

```bash
sudo journalctl -u mariadb --no-pager -n 50
```

### Access denied for user 'ejbca'

Verify the user exists and has the right privileges:

```bash
sudo mysql -u root -p -e "SHOW GRANTS FOR 'ejbca'@'localhost';"
```

### Key too long error (utf8mb4 index limit)

If you see `Specified key was too long; max key length is 767 bytes` during EJBCA deployment, this is a known issue with `utf8mb4`. The reason: `utf8mb4` uses 4 bytes per character, so a `VARCHAR(255)` column requires 1,020 bytes for its index — exceeding the 767-byte InnoDB index limit. The workaround is to use `utf8` (which uses 3 bytes per character, so `VARCHAR(255)` = 765 bytes — just under the limit):

```bash
sudo mariadb -u root -p -e "DROP DATABASE ejbca; CREATE DATABASE ejbca CHARACTER SET utf8 COLLATE utf8_unicode_ci;"
sudo mariadb -u root -p -e "GRANT ALL PRIVILEGES ON ejbca.* TO 'ejbca'@'localhost'; GRANT ALL PRIVILEGES ON ejbca.* TO 'ejbca'@'127.0.0.1'; FLUSH PRIVILEGES;"
```

Then re-run the EJBCA deployment from Module 05.

---

**Next Module:** [04 - WildFly 35 Setup](04_ejbca_wildfly.md)

---

*[← Previous: Module 02 - Configurations](02_ejbca_configurations.md) | [Next: Module 04 - WildFly →](04_ejbca_wildfly.md)*
