# FreeIPA Deployment and User Provisioning on RedHat Server

## Overview

This document outlines the deployment of a FreeIPA server on a RedHat Enterprise Linux (RHEL) 9 environment for LDAP-based identity management, designed to integrate with SailPoint IdentityIQ for user lifecycle automation. It covers server preparation, FreeIPA installation, user provisioning, and verification.

---

## 1. Environment Preparation

- **Operating System**: RedHat Enterprise Linux 9 (duplicate of the production SailPoint host).
- **Purpose**: Dedicated FreeIPA server for LDAP/identity management to avoid port conflicts with SailPoint.
- **Ports Required**:

| Protocol | Ports                     |
|----------|---------------------------|
| TCP      | 80, 443, 389, 636, 88, 464, 53 |
| UDP      | 88, 464, 53, 123          |

### Steps
1. Ensure the system is updated:
   ```bash
   sudo dnf update -y
   ```
2. Verify no conflicting services (e.g., OpenLDAP) are installed:
   ```bash
   sudo dnf remove -y openldap-servers openldap-clients
   sudo systemctl stop slapd
   sudo rm -rf /var/lib/ldap/* /etc/openldap/slapd.d/*
   ```

---

## 2. DNS and Hostname Configuration

- **Hostname**: `ipa.demo.local`
- **Static IP**: `192.168.75.8`
- **DNS Forwarders**: Not required for standalone environment (`--no-forwarders`).

### Steps
1. Set hostname:
   ```bash
   sudo hostnamectl set-hostname ipa.demo.local
   ```
2. Configure static IP (replace `<connection_name>` with your network interface, e.g., `eth0`):
   ```bash
   sudo nmcli connection modify <connection_name> ipv4.addresses 192.168.75.8/24
   sudo nmcli connection modify <connection_name> ipv4.gateway <gateway_ip>
   sudo nmcli connection modify <connection_name> ipv4.dns 192.168.75.8
   sudo nmcli connection modify <connection_name> ipv4.method manual
   sudo nmcli connection up <connection_name>
   ```
3. Update `/etc/hosts`:
   ```bash
   echo "192.168.75.8 ipa.demo.local ipa" | sudo tee -a /etc/hosts
   ```
4. Verify:
   ```bash
   hostname
   ping -c 3 ipa.demo.local
   ```

---

## 3. FreeIPA Server Installation

Install and configure FreeIPA using the interactive installer in unattended mode.

### Steps
1. Install FreeIPA:
   ```bash
   sudo dnf install -y ipa-server ipa-server-dns
   ```
2. Run the installer:
   ```bash
   sudo ipa-server-install \
     --unattended \
     --realm=DEMO.LOCAL \
     --domain=demo.local \
     --admin-password='SailPoint123!' \
     --ds-password='SailPoint123!' \
     --setup-dns \
     --no-forwarders
   ```
3. Verify installation:
   ```bash
   ipa ping
   sudo systemctl status ipa
   ```

### Key Notes
- **Components Installed**:
  - Standalone Certificate Authority (Dogtag)
  - Directory Server (389/636)
  - Kerberos KDC (88/464)
  - Web Interface (80/443)
  - DNS (bind)
- **Backup**: CA certificates are stored in `/root/cacert.p12`.
- **Firewall**: Ensure required ports are open:
  ```bash
  sudo firewall-cmd --add-port={80/tcp,443/tcp,389/tcp,636/tcp,88/tcp,464/tcp,53/tcp,88/udp,464/udp,53/udp,123/udp} --permanent
  sudo firewall-cmd --reload
  ```

---

## 4. FreeIPA User Creation Script

Create 20 realistic users in FreeIPA with a common password for testing.

### Script: `create_freeipa_users.sh`
```bash
#!/usr/bin/env bash
# create_freeipa_users.sh
# Create 20 realistic users in FreeIPA
# Run as a user with a Kerberos ticket (kinit admin)

set -euo pipefail
IFS=$'\n\t'

# Configuration
DEFAULT_PASSWORD="SailPoint123!"
REALM="DEMO.LOCAL"
DOMAIN="demo.local"

# Realistic names (20)
FIRST_NAMES=(Olivia Liam Emma Noah Ava William Sophia James Isabella Benjamin Mia Lucas Charlotte Mason Amelia Ethan Harper Oliver Evelyn Jacob)
LAST_NAMES=(Smith Johnson Williams Brown Jones Garcia Miller Davis Rodriguez Martinez Hernandez Lopez Gonzalez Wilson Anderson Thomas Taylor Moore Jackson Martin)

# Sanity check: array length
if [ "${#FIRST_NAMES[@]}" -ne "${#LAST_NAMES[@]}" ]; then
  echo "ERROR: first/last name arrays length mismatch" >&2
  exit 2
fi

echo "Creating ${#FIRST_NAMES[@]} users in FreeIPA (domain: ${DOMAIN})."
echo "Ensure you have a Kerberos ticket: run 'kinit admin' first."
read -rp "Proceed? [y/N] " RESP
if [[ "${RESP,,}" != "y" ]]; then
  echo "Aborted by user."
  exit 0
fi

# Obtain Kerberos ticket
echo "SailPoint123!" | kinit admin

CREATED=0
SKIPPED=0
FAILED=0

for i in "${!FIRST_NAMES[@]}"; do
  FN="${FIRST_NAMES[$i]}"
  LN="${LAST_NAMES[$i]}"
  USER_UID="$(printf "%s.%s" "${FN,,}" "${LN,,}")"
  USER_UID="${USER_UID// /}"
  EMAIL="${FN,,}.${LN,,}@example.com"

  echo "----"
  echo "User: ${FN} ${LN}"
  echo "UID : ${USER_UID}"
  echo "Email: ${EMAIL}"

  # Check if user exists
  if ipa user-find "${USER_UID}" >/dev/null 2>&1; then
    echo "-> SKIP: user ${USER_UID} already exists"
    SKIPPED=$((SKIPPED+1))
    continue
  fi

  # Create user
  if printf "%s\n%s\n" "${DEFAULT_PASSWORD}" "${DEFAULT_PASSWORD}" | \
      ipa user-add "${USER_UID}" --first="${FN}" --last="${LN}" --email="${EMAIL}" --password >/dev/null 2>&1; then
    echo "-> CREATED: ${USER_UID}"
    CREATED=$((CREATED+1))
  else
    echo "-> FAILED to create ${USER_UID}"
    FAILED=$((FAILED+1))
  fi
done

echo "----"
echo "Summary: created=${CREATED}, skipped=${SKIPPED}, failed=${FAILED}"
echo "Verify users with:"
echo "  ipa user-show ${FIRST_NAMES[0],,}.${LAST_NAMES[0],,}"
echo "  ipa user-find"
```

### Steps to Run
1. Save the script:
   ```bash
   nano create_freeipa_users.sh
   ```
   Paste the script, save, and exit.
2. Make executable:
   ```bash
   chmod +x create_freeipa_users.sh
   ```
3. Run the script:
   ```bash
   kinit admin  # Enter password: SailPoint123!
   ./create_freeipa_users.sh
   ```

### Notes
- **Kerberos Ticket**: The script requires a valid Kerberos ticket (`kinit admin`).
- **Password**: All users are created with `SailPoint123!` for testing. Apply password policies later in FreeIPA.
- **User Verification**:
  ```bash
  ipa user-find
  ipa user-show olivia.smith
  ```

---

## 5. Verification

### Check FreeIPA Status
```bash
ipa ping
sudo systemctl status ipa
```

### List Users
```bash
ipa user-find
```
Expected: Lists 20 users (e.g., `olivia.smith`, `liam.johnson`, etc.).

### LDAP Connectivity Test
```bash
ldapsearch -x -H ldap://ipa.demo.local:389 -D "uid=admin,cn=users,cn=accounts,dc=demo,dc=local" -W -b "dc=demo,dc=local" "(objectClass=posixAccount)"
```
- Enter password: `SailPoint123!`
- Expected: Lists all users under `cn=users,cn=accounts,dc=demo,dc=local`.

Login WebUI Dashboard 

<img width="468" height="240" alt="image" src="https://github.com/user-attachments/assets/9ecf6138-f12a-4599-b8f1-0757e43e72a9" />


---

## 6. Next Steps
1. **SailPoint Integration**:
   - Configure SailPoint Identityiiq to connect to FreeIPA:
     - **LDAP URL**: `ldap://ipa.demo.local:389` or `ldaps://ipa.demo.local:636`
     - **Bind DN**: `uid=admin,cn=users,cn=accounts,dc=demo,dc=local`
     - **Password**: `SailPoint123!`
     - **Base DN**: `cn=users,cn=accounts,dc=demo,dc=local`
     - **Object Class**: `posixAccount`
     - **UID Attribute**: `uid`
2. **User Lifecycle Management**: Set up provisioning, de-provisioning, and attribute synchronization in SailPoint.
3. **Security**: Enable Kerberos authentication and configure password policies in FreeIPA.
4. **Backup**: Export CA and configuration:
   ```bash
   cp /root/cacert.p12 /path/to/backup/
   ```

---

## 7. References
- [FreeIPA Documentation](https://www.freeipa.org/page/Documentation)
- [SailPoint Identityiiq LDAP Integration Guide](https://documentation.sailpoint.com/identitynow)

