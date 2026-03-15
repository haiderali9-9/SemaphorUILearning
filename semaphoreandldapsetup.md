# OpenLDAP Server Setup with Semaphore Integration

## Table of Contents
- [Step 1: Setup Domain (DNS)](#step-1-setup-domain-dns)
- [Step 2: Install Packages](#step-2-install-packages)
- [Step 3: Reconfigure slapd with Domain](#step-3-reconfigure-slapd-with-domain)
- [Step 4: Create Organization Units (OU)](#step-4-create-organization-units-ou)
- [Step 5: Create Users](#step-5-create-users)
- [Step 6: Semaphore Setup](#step-6-semaphore-setup)

---

## Step 1: Setup Domain (DNS)

Go to your domain provider and add an **A Record**:

| Type | Name | Value |
|------|------|-------|
| A Record | `corp` | `YOUR_SERVER_IP` |

This will make `corp.devopsbyhaiderali.site` point to your server.

Verify DNS is working:
```bash
ping corp.devopsbyhaiderali.site
```

---

## Step 2: Install Packages

```bash
# Update system
sudo apt update

# Install slapd and ldap-utils
sudo apt install slapd ldap-utils -y
```

During installation it will ask for an **admin password** — set a strong password and remember it.

---

## Step 3: Reconfigure slapd with Domain

```bash
sudo dpkg-reconfigure slapd
```

Answer the questions as follows:

| Question | Answer |
|----------|--------|
| Omit OpenLDAP server configuration? | **No** |
| DNS domain name? | `corp.devopsbyhaiderali.site` |
| Organization name? | `DevOpsCorp` |
| Administrator password? | Your strong password |
| Confirm password? | Same password |
| Remove database when slapd is purged? | **No** |
| Move old database? | **Yes** |

Verify LDAP is working:
```bash
ldapsearch -x -H ldap://localhost \
  -b "dc=corp,dc=devopsbyhaiderali,dc=site" \
  -D "cn=admin,dc=corp,dc=devopsbyhaiderali,dc=site" \
  -W
```

Expected output:
```
result: 0 Success
numEntries: 1
```

---

## Step 4: Create Organization Units (OU)

Create the base structure file:
```bash
nano base.ldif
```

Paste the following:
```ldif
dn: ou=users,dc=corp,dc=devopsbyhaiderali,dc=site
objectClass: organizationalUnit
ou: users

dn: ou=groups,dc=corp,dc=devopsbyhaiderali,dc=site
objectClass: organizationalUnit
ou: groups
```

Apply the base structure:
```bash
ldapadd -x \
  -D "cn=admin,dc=corp,dc=devopsbyhaiderali,dc=site" \
  -W -f base.ldif
```

Verify OUs were created:
```bash
ldapsearch -x -H ldap://localhost \
  -b "dc=corp,dc=devopsbyhaiderali,dc=site" \
  -D "cn=admin,dc=corp,dc=devopsbyhaiderali,dc=site" \
  -W
```

---

## Step 5: Create Users

### Generate Password Hash

```bash
slappasswd
```

Copy the generated hash e.g. `{SSHA}ovrBiNQOw4ljeGXhrIqZE9elpEIGB0Z9`

### Create User File

```bash
nano adduser.ldif
```

Paste the following (replace values as needed):
```ldif
dn: uid=john,ou=users,dc=corp,dc=devopsbyhaiderali,dc=site
objectClass: inetOrgPerson
objectClass: posixAccount
objectClass: shadowAccount
uid: john
sn: Doe
givenName: John
cn: John Doe
mail: john@corp.devopsbyhaiderali.site
uidNumber: 10001
gidNumber: 10001
homeDirectory: /home/john
loginShell: /bin/bash
userPassword: {SSHA}ovrBiNQOw4ljeGXhrIqZE9elpEIGB0Z9
```

### Add User to LDAP

```bash
ldapadd -x \
  -D "cn=admin,dc=corp,dc=devopsbyhaiderali,dc=site" \
  -W -f adduser.ldif
```

### Verify User was Added

```bash
ldapsearch -x -H ldap://localhost \
  -b "ou=users,dc=corp,dc=devopsbyhaiderali,dc=site" \
  -D "cn=admin,dc=corp,dc=devopsbyhaiderali,dc=site" \
  -W "(uid=john)"
```

### Test User Authentication

```bash
ldapsearch -x -H ldap://localhost \
  -b "dc=corp,dc=devopsbyhaiderali,dc=site" \
  -D "uid=john,ou=users,dc=corp,dc=devopsbyhaiderali,dc=site" \
  -W
```

---

## Step 6: Semaphore Setup

### Open Firewall Port

```bash
sudo ufw allow 389/tcp
sudo ufw reload
```

### Docker Compose File

Create the docker-compose file:
```bash
nano docker-compose.yml
```

Paste the following:
```yaml
services:
  mysql:
    restart: unless-stopped
    image: mysql:8.0.36
    hostname: mysql
    volumes:
      - semaphore-mysql:/var/lib/mysql
    environment:
      MYSQL_RANDOM_ROOT_PASSWORD: 'yes'
      MYSQL_DATABASE: semaphore
      MYSQL_USER: semaphore
      MYSQL_PASSWORD: semaphore

  semaphore:
    restart: unless-stopped
    ports:
      - 3000:3000
    image: semaphoreui/semaphore:v2.9.37
    environment:
      # Database Settings
      SEMAPHORE_DB_USER: semaphore
      SEMAPHORE_DB_PASS: semaphore
      SEMAPHORE_DB_HOST: mysql
      SEMAPHORE_DB_PORT: 3306
      SEMAPHORE_DB_DIALECT: mysql
      SEMAPHORE_DB: semaphore
      # Semaphore Settings
      SEMAPHORE_PLAYBOOK_PATH: /tmp/semaphore/
      SEMAPHORE_ADMIN_PASSWORD: 'your_admin_password'
      SEMAPHORE_ADMIN_NAME: admin
      SEMAPHORE_ADMIN_EMAIL: admin@localhost
      SEMAPHORE_ADMIN: admin
      SEMAPHORE_ACCESS_KEY_ENCRYPTION: gs72mPntFATGJs9qK0pQ0rKtfidlexiMjYCH9gWKhTU=
      # LDAP Settings
      SEMAPHORE_LDAP_ACTIVATED: 'yes'
      SEMAPHORE_LDAP_HOST: corp.devopsbyhaiderali.site
      SEMAPHORE_LDAP_PORT: '389'
      SEMAPHORE_LDAP_NEEDTLS: 'no'
      SEMAPHORE_LDAP_DN_BIND: 'cn=admin,dc=corp,dc=devopsbyhaiderali,dc=site'
      SEMAPHORE_LDAP_PASSWORD: 'your_ldap_admin_password'
      SEMAPHORE_LDAP_DN_SEARCH: 'ou=users,dc=corp,dc=devopsbyhaiderali,dc=site'
      SEMAPHORE_LDAP_SEARCH_FILTER: "(&(uid=%s))"
      TZ: UTC
    depends_on:
      - mysql

volumes:
  semaphore-mysql:
```

> **Note:** Replace `your_admin_password` and `your_ldap_admin_password` with your actual passwords.

### Start Semaphore

```bash
docker compose up -d
```

### Check Logs

```bash
docker compose logs -f semaphore
```

### Access Semaphore

Open your browser and go to:
```
http://your-server-ip:3000
```

Login with:

| Method | Username | Password |
|--------|----------|----------|
| Local Admin | `admin` | your_admin_password |
| LDAP User | `john` | password set with slappasswd |

### SSH Tunnel (Optional)

If accessing from a remote machine:
```bash
ssh -L 3000:localhost:3000 user@your-server-ip
```

Then open `http://localhost:3000` in your browser.
