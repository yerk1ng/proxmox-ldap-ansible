# Ansible Role: Proxmox LDAP & FreeIPA Integration (PVE & PBS)

[![License: MIT](https://img.shields.io/badge/License-MIT-blue.svg)](LICENSE)
[![Ansible](https://img.shields.io/badge/Ansible-2.12+-black.svg)](https://www.ansible.com/)
[![Proxmox VE](https://img.shields.io/badge/Proxmox%20VE-8.x%20%2F%209.x-orange.svg)](https://www.proxmox.com/en/proxmox-virtual-environment)
[![Proxmox Backup Server](https://img.shields.io/badge/Proxmox%20Backup%20Server-3.x+-red.svg)](https://www.proxmox.com/en/proxmox-backup-server)

An enterprise-ready, idempotent Ansible role to automate **LDAP / FreeIPA** authentication realms, user/group synchronization, automated calendar-based sync jobs, and ACL permission assignments for both **Proxmox Virtual Environment (PVE)** and **Proxmox Backup Server (PBS)**.

---

## Why This Role?

Most community Ansible modules and roles for Proxmox LDAP either:
1. Only support Proxmox VE and completely ignore **Proxmox Backup Server (PBS)**.
2. Require pre-existing Proxmox REST API tokens and expose open API ports.
3. Only add the basic realm and leave users stranded with zero permissions, requiring manual Web UI clicks to map ACLs or configure synchronization jobs.

### Key Capabilities

* **Dual Target Architecture (PVE & PBS):** Unified role managing authentication across your compute hypervisors (PVE) and dedicated backup storage servers (PBS).
* **Automated Realm Sync Jobs (`/etc/pve/jobs.cfg`):** Deploys persistent, scheduled sync tasks so new directory users and group memberships are automatically synchronized without manual intervention.
* **Instant ACL & Permission Mapping:** Automatically creates and associates administrative permissions (`Administrator` role on `/`) with your synced directory group or users upon deployment.
* **Proxmox Backup Server Idempotency:** Implements complete management for `/etc/proxmox-backup/domains.cfg` via `proxmox-backup-manager`, performing attribute synchronization and automatic ACL assignment.
* **Cluster-Aware Execution:** In PVE clusters, cluster-wide changes (`/etc/pve/`) are intelligently executed on a single cluster gateway node, preventing `pmxcfs` lock contention.
* **Zero External API Tokens Required:** Operates cleanly through native SSH and local Proxmox management utilities (`pveum` and `proxmox-backup-manager`).

---

## Architecture Flow

```
                     ┌────────────────────────────────┐
                     │    FreeIPA / LDAP Directory    │
                     └───────────────┬────────────────┘
                                     │ LDAPS (Port 636)
               ┌─────────────────────┴─────────────────────┐
               ▼                                           ▼
┌─────────────────────────────┐             ┌─────────────────────────────┐
│    Proxmox VE Cluster       │             │   Proxmox Backup Server     │
│                             │             │                             │
│ 1. pveum realm add/modify   │             │ 1. proxmox-backup-manager   │
│ 2. /etc/pve/jobs.cfg (sync) │             │    ldap create/update       │
│ 3. pveum acl modify (Group) │             │ 2. sync realm               │
│ 4. Automatic User Sync      │             │ 3. Assign Admin ACL on /    │
└─────────────────────────────┘             └─────────────────────────────┘
```

---

## Requirements

* **Proxmox Virtual Environment:** 8.x or 9.x
* **Proxmox Backup Server:** 3.x or newer
* **Directory Service:** FreeIPA, Red Hat IdM, OpenLDAP, or 389 Directory Server.
* **Ansible:** 2.12 or newer installed on the control node.

---

## Role Variables

All default variables are documented in [`defaults/main.yml`](defaults/main.yml).

### General LDAP Connection

| Variable | Default | Description |
|---|---|---|
| `proxmox_ldap_realm` | `freeipa` | Name of the authentication realm in Proxmox. |
| `proxmox_ldap_comment` | `LDAP / FreeIPA Realm...` | Description displayed in the Web UI login dropdown. |
| `proxmox_ldap_default` | `true` | Whether this realm is selected by default on the login screen. |
| `proxmox_ldap_server1` | `ldap1.example.com` | Primary LDAP server hostname or IP. |
| `proxmox_ldap_server2` | `ldap2.example.com` | Secondary / failover LDAP server. |
| `proxmox_ldap_port` | `636` | LDAP port (usually 636 for LDAPS, 389 for StartTLS/Plain). |
| `proxmox_ldap_mode` | `ldaps` | Connection security mode (`ldaps`, `starttls`, or `plain`). |
| `proxmox_ldap_base_dn` | `cn=accounts,dc=example,dc=com` | Base DN for directory searches. |
| `proxmox_ldap_bind_dn` | `uid=pve_sa,cn=users,...` | Bind service account DN. |
| `proxmox_ldap_bind_password` | `""` | Password for the bind service account (use Ansible Vault). |
| `proxmox_ldap_user_attr` | `uid` | Attribute holding the login username (e.g. `uid` or `sAMAccountName`). |

### Filtering & Attribute Synchronization

| Variable | Default | Description |
|---|---|---|
| `proxmox_ldap_ipa_group` | `pve_admins` | Target group name in LDAP that receives Proxmox admin privileges. |
| `proxmox_ldap_filter` | `(memberOf=cn={{ ipa_group }},...)` | LDAP search filter restricting which users can log in. |
| `proxmox_ldap_sync_attributes` | `email=mail,firstname=givenName,lastname=sn` | Attribute mapping for user profiles in Proxmox. |
| `proxmox_ldap_group_filter` | `(cn={{ ipa_group }})` | Filter used when querying directory groups. |
| `proxmox_ldap_group_classes` | `ipausergroup,groupOfNames` | Object classes representing groups. |

### PVE Realm Sync Job (`/etc/pve/jobs.cfg`)

| Variable | Default | Description |
|---|---|---|
| `proxmox_ldap_sync_schedule` | `21:00` | Proxmox calendar event schedule for automated sync. |
| `proxmox_ldap_sync_scope` | `both` | Synchronization scope (`users`, `groups`, or `both`). |
| `proxmox_ldap_sync_enable_new` | `1` | Automatically enable newly discovered users. |
| `proxmox_ldap_sync_remove_vanished` | `acl;entry;properties` | Cleanup actions when a user/group is deleted in LDAP. |

### Access Control (ACL)

| Variable | Default | Description |
|---|---|---|
| `proxmox_ldap_manage_acl` | `true` | Whether to automatically configure group/user ACLs. |
| `proxmox_ldap_admin_role` | `Administrator` | Proxmox role granted to the administrator group. |
| `proxmox_ldap_admin_path` | `/` | Target Proxmox ACL path (e.g. `/` for cluster-wide admin). |

---

## Directory Service Preparation (FreeIPA Example)

Create a dedicated read-only service account for Proxmox in FreeIPA:

```bash
# 1. Create the service account
ipa user-add pve_sa \
    --first="Proxmox" \
    --last="ServiceAccount" \
    --password

# 2. (Optional) Create the administrators group
ipa group-add pve_admins --desc="Proxmox Cluster Administrators"

# 3. Add admin users to the group
ipa group-add-member pve_admins --users=alice,bob
```

---

## Quick Start Example

### 1. Inventory (`inventory.ini`)

Group your hypervisors and backup servers:

```ini
[proxmox:children]
proxmox_ve
proxmox_backup

[proxmox_ve]
pve-01.example.com
pve-02.example.com
pve-03.example.com

[proxmox_backup]
pbs-01.example.com
```

### 2. Playbook (`playbook.yml`)

```yaml
- name: Deploy LDAP Authentication to Proxmox
  hosts: proxmox
  become: true
  roles:
    - role: proxmox_ldap
      vars:
        proxmox_ldap_realm: "freeipa"
        proxmox_ldap_server1: "ipa1.example.com"
        proxmox_ldap_server2: "ipa2.example.com"
        proxmox_ldap_base_dn: "cn=accounts,dc=example,dc=com"
        proxmox_ldap_bind_dn: "uid=pve_sa,cn=users,cn=accounts,dc=example,dc=com"
        proxmox_ldap_bind_password: "SecretBindPasswordHere"
        proxmox_ldap_ipa_group: "pve_admins"
        proxmox_ldap_sync_schedule: "02:00"
```

### 3. Run the Playbook

```bash
ansible-playbook -i inventory.ini playbook.yml
```

---

## Verification

### On Proxmox VE (PVE)
Check realm configuration, sync job, and ACL permissions:
```bash
# Verify realm exists
pveum realm list

# Verify scheduled sync job
cat /etc/pve/jobs.cfg

# Manually trigger immediate sync test
pveum realm sync freeipa --scope both

# Check ACLs
pveum acl list
```

### On Proxmox Backup Server (PBS)
Check realm configuration and user permissions:
```bash
# Verify realm
proxmox-backup-manager ldap list

# Verify synced users
proxmox-backup-manager user list

# Verify ACLs
proxmox-backup-manager acl list
```

---

## License

MIT License. See [LICENSE](LICENSE) for full details.

## Author

**Yerkin Nurekenov** ([GitHub: @yerk1ng](https://github.com/yerk1ng))
