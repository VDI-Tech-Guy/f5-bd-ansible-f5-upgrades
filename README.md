# f5-bd-ansible-f5-upgrades

Automated upgrade orchestration for F5OS (rSeries / Velos) and BIG-IP devices (standalone and HA pairs), with pre/post snapshot collection and JSON diff reporting.

---

## Repository Structure

```
f5-bd-ansible-f5-upgrades/
├── ansible.cfg                         # Ansible configuration
├── galaxy.yml                          # Collection metadata
├── collections/
│   └── requirements.yml                # Collection dependencies
├── inventory/
│   └── hosts.ini                       # Example inventory (copy and populate)
├── group_vars/
│   ├── bigip.yml                       # Shared BIG-IP variables
│   └── f5os.yml                        # Shared F5OS variables
├── host_vars/                          # Per-device variable overrides
├── snapshots/                          # Snapshot JSON output (git-ignored)
├── playbooks/
│   ├── BIG-IP/
│   │   └── Upgrades/
│   │       ├── standalone/
│   │       │   ├── upgrade.yaml
│   │       │   └── vars/upgrade_vars.yml
│   │       └── ha/
│   │           ├── upgrade.yaml
│   │           └── vars/upgrade_vars.yml
│   ├── F5OS/
│   │   └── Upgrades/
│   │       ├── rSeries/
│   │       │   ├── upgrade.yaml
│   │       │   └── vars/upgrade_vars.yml
│   │       └── Velos/
│   │           ├── upgrade.yaml
│   │           └── vars/upgrade_vars.yml
│   └── Common/
│       └── Snapshots/                  # Standalone snapshot playbooks
└── roles/
    ├── bigip_wait/                     # Layered post-reboot readiness wait
    ├── bigip_upgrade/                  # BIG-IP image upload + install + reboot
    ├── snapshot_pre/                   # Pre-upgrade state capture
    ├── snapshot_post/                  # Post-upgrade state capture + diff
    ├── snapshot_report/                # Stdout summary of diff results
    └── f5os_upgrade/
        ├── rseries/                    # F5OS rSeries image import + upgrade
        └── velos/                      # F5OS Velos controller + partition upgrade
```

---

## Requirements

### Ansible
- Ansible Core >= 2.12
- Ansible Automation Platform 2.x (if running via AAP)

### Collections
Install all required collections:
```bash
ansible-galaxy collection install -r collections/requirements.yml
```

| Collection | Purpose | Min Version |
|---|---|---|
| `f5networks.f5_bigip` | BIG-IP iControl REST modules | >= 1.0.0 |
| `f5networks.f5os` | F5OS REST modules | >= 1.0.0 |
| `f5networks.f5_modules` | Legacy BIG-IP modules (13.x fallback) | >= 1.0.0 |
| `ansible.utils` | JSON diff / IP address filters | >= 2.0.0 |
| `ansible.posix` | File/copy utilities | >= 1.0.0 |
| `ansible.netcommon` | Network connection plugins | >= 2.0.0 |
| `community.general` | General utilities | >= 4.0.0 |

### BIG-IP Compatibility
- BIG-IP **13.1+** (v13 floor — uses `/mgmt/shared/echo` for REST readiness, not `/mgmt/tm/sys/ready`)

### F5OS Compatibility
- F5OS-A (rSeries) — tested on 1.x+
- F5OS-C (Velos) — tested on 1.x+

---

## Quick Start

### 1. Clone and configure inventory
```bash
git clone https://github.com/f5devcentral/f5-bd-ansible-f5-upgrades
cd f5-bd-ansible-f5-upgrades
cp inventory/hosts.ini inventory/my_hosts.ini
# Edit my_hosts.ini with your device IPs and groups
```

### 2. Install collections
```bash
ansible-galaxy collection install -r collections/requirements.yml
```

### 3. Stage software images
Place ISO/image files on your Ansible controller at the path set in `bigip_image_local_path` (default `/mnt/software/bigip`) or `f5os_image_local_path` (default `/mnt/software/f5os`).

### 4. Run an upgrade

**BIG-IP Standalone:**
```bash
ansible-playbook playbooks/BIG-IP/Upgrades/standalone/upgrade.yaml \
  -i inventory/my_hosts.ini \
  -e "bigip_target_version=17.1.1 bigip_image_filename=BIGIP-17.1.1-0.0.4.iso" \
  --limit bigip-standalone-01
```

**BIG-IP HA Pair:**
```bash
ansible-playbook playbooks/BIG-IP/Upgrades/ha/upgrade.yaml \
  -i inventory/my_hosts.ini \
  -e "bigip_target_version=17.1.1 bigip_image_filename=BIGIP-17.1.1-0.0.4.iso" \
  --limit bigip_ha_pair01
```

**F5OS rSeries:**
```bash
ansible-playbook playbooks/F5OS/Upgrades/rSeries/upgrade.yaml \
  -i inventory/my_hosts.ini \
  -e "f5os_image_filename=F5OS-A-1.8.0-13798.R4R5.iso" \
  --limit rseries-r2600-01
```

**F5OS Velos:**
```bash
ansible-playbook playbooks/F5OS/Upgrades/Velos/upgrade.yaml \
  -i inventory/my_hosts.ini \
  -e "f5os_image_filename=F5OS-C-1.8.0-3198.CHASSIS.iso" \
  --limit velos-chassis-01-ctrl:velos-chassis-01-part1
```

---

## Snapshot & Diff

Every upgrade run produces a JSON diff file in the `snapshots/` directory:
```
snapshots/
  <hostname>_<timestamp>_pre.json
  <hostname>_<timestamp>_post.json
  <hostname>_<timestamp>_diff.json
```

### What is captured

| Data Point | BIG-IP | F5OS |
|---|:---:|:---:|
| Virtual server status (up/down) | ✅ | — |
| Pool member health | ✅ | — |
| Route table | ✅ | — |
| ARP table | ✅ | — |
| Running config hash | ✅ | — |
| UCS backup | ✅ | — |
| Tenant status | — | ✅ |
| Platform software version | ✅ | ✅ |

### Snapshot-only run (no upgrade)
```bash
# Pre-snapshot only
ansible-playbook playbooks/BIG-IP/Upgrades/standalone/upgrade.yaml \
  --tags pre_snapshot --limit bigip-standalone-01

# Post-snapshot + diff only (after manual upgrade)
ansible-playbook playbooks/BIG-IP/Upgrades/standalone/upgrade.yaml \
  --tags post_snapshot --limit bigip-standalone-01
```

---

## BIG-IP Reboot Wait Strategy

The `bigip_wait` role uses a layered approach to avoid false-positive readiness (TCP port up but REST not ready):

```
1. Wait for SSH port 22 to go DOWN  → confirms reboot actually started
2. Wait for SSH port 22 to come UP  → kernel + SSH stack running
3. Wait for HTTPS port 443 to come UP → web stack listening
4. Poll /mgmt/shared/echo            → returns {"stage":"STARTED"} only
                                        when REST is fully initialised
                                        (works BIG-IP 13.1+)
5. Optional mcpd check via tmsh      → confirms config sync is ready
```

Connection drops during reboot are handled via `ignore_errors: true` on the reboot trigger task with all wait tasks delegated to `localhost`.

---

## Variables Reference

### BIG-IP Upgrade Variables

| Variable | Default | Description |
|---|---|---|
| `bigip_target_version` | — | Version to verify post-upgrade |
| `bigip_image_filename` | — | ISO filename in `bigip_image_local_path` |
| `bigip_image_local_path` | `/mnt/software/bigip` | Controller path to image staging dir |
| `bigip_install_volume` | `HD1.2` | Boot volume to install to |
| `bigip_reboot_after_install` | `true` | Reboot after install |
| `bigip_upgrade_dry_run` | `false` | Skip reboot (stage only) |
| `bigip_reboot_initial_delay` | `60` | Seconds before starting port probes |
| `bigip_wait_retry_delay` | `20` | Seconds between REST probe retries |
| `bigip_wait_max_retries` | `30` | Max REST probe attempts (~10 min) |
| `bigip_failback` | `false` | (HA only) Fail back after upgrade |

### F5OS Upgrade Variables

| Variable | Default | Description |
|---|---|---|
| `f5os_image_filename` | — | ISO filename in `f5os_image_local_path` |
| `f5os_image_local_path` | `/mnt/software/f5os` | Controller path to image staging dir |
| `f5os_set_next_boot` | `true` | Set staged image as next boot |
| `f5os_reboot_after_stage` | `true` | Reboot after staging |
| `f5os_upgrade_dry_run` | `false` | Skip reboot (stage only) |
| `f5os_reboot_initial_delay` | `90` | Seconds before starting probes |
| `f5os_wait_retry_delay` | `30` | Seconds between probe retries |
| `f5os_wait_max_retries` | `40` | Max probe attempts (~20 min) |
| `f5os_velos_upgrade_controller_first` | `true` | (Velos) Upgrade controller before partitions |

---

## Development Roadmap

- [x] Repo scaffold, inventory, group_vars, playbooks
- [ ] `bigip_wait` role — layered reboot wait tasks
- [ ] `snapshot_pre` role — baseline state capture tasks
- [ ] `snapshot_post` role — post-upgrade capture + JSON diff tasks
- [ ] `snapshot_report` role — stdout diff summary
- [ ] `bigip_upgrade` role — image upload + install + reboot tasks
- [ ] `f5os_upgrade/rseries` role — rSeries image import + upgrade tasks
- [ ] `f5os_upgrade/velos` role — Velos controller + partition upgrade tasks
- [ ] AAP Job Template examples
- [ ] Vault-encrypted credential examples

---

## License

Apache 2.0 — see LICENSE file.
