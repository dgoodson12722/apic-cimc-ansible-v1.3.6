# Cisco APIC CIMC Upgrade Framework

**Release:** 1.3.6  
**Status:** Functional operator-assisted release, aligned to Cisco upgrade guidance listed below

## Overview

This Ansible project validates and assists Cisco APIC CIMC upgrades. It provides
Redfish connectivity testing, hardware and firmware inventory, APIC cluster
health checks, a local compatibility decision, Cisco-image checksum validation,
operator-controlled upgrade pauses, recovery monitoring, and before/after JSON
reports.

The disruptive CIMC upgrade remains under engineer control. Ansible performs the
prechecks, displays the applicable Cisco procedure, pauses, and resumes only
after the engineer confirms completion. This repository prepares CIMC for an
APIC software upgrade; it does **not** initiate the APIC software upgrade itself.

## Cisco Authoritative References

The workflow in release 1.3.1 was aligned and re-verified on **August 11, 2026** against the following Cisco documentation. Cisco documentation can change after this repository is released, so the engineer must re-check the current Cisco documentation and the Release Notes for the exact source and target ACI releases before every production maintenance window.

1. **[Upgrade CIMC on APIC — Cisco Document ID 215178](https://www.cisco.com/c/en/us/support/docs/cloud-systems-management/application-policy-infrastructure-controller-apic/215178-upgrade-cimc-on-apic.html)**  
   Cisco document updated **July 3, 2025**. This is the primary reference for the legacy CIMC/HUU procedure used by this framework. It covers determining the required CIMC version from the APIC Release Notes, obtaining the Cisco software image, validating the Cisco-published MD5 checksum, and performing the HUU/vKVM-based CIMC upgrade on APIC UCS C-Series servers.

2. **[Cisco APIC Installation and ACI Upgrade and Downgrade Guide](https://www.cisco.com/c/en/us/td/docs/dcn/aci/apic/all/apic-installation-aci-upgrade-downgrade/Cisco-APIC-Installation-ACI-Upgrade-Downgrade-Guide/g-aci-firmware-upgrade-overview/workflow-to-upgrade-downgrade-apic.html)**  
   Cisco guide updated **July 12, 2026**. This is the primary reference for the overall ACI upgrade sequence, compatibility and backup prerequisites, APIC cluster requirements, CIMC/HUU placement in the upgrade workflow, APIC controller upgrades, and the requirement that the APIC cluster be fully fit before proceeding to switch upgrades.

3. **[Upgrade or downgrade the Cisco APIC CIMC from releases 6.2 or later](https://www.cisco.com/c/en/us/td/docs/dcn/aci/apic/all/apic-installation-aci-upgrade-downgrade/Cisco-APIC-Installation-ACI-Upgrade-Downgrade-Guide/g-upgrading-or-downgrading-with-apic-release-62-or-later-using-the-gui/upgrading-or-downgrading-the-cisco-apic-cimc-from-releases-62-or-later.html)**  
   Cisco page updated **July 12, 2026**. This is the specific authoritative reference for the APIC-integrated, orchestrated CIMC workflow introduced in ACI 6.2(1) and later.

4. **[Pre-Upgrade/Downgrade Checklists](https://www.cisco.com/c/en/us/td/docs/dcn/aci/apic/all/apic-installation-aci-upgrade-downgrade/Cisco-APIC-Installation-ACI-Upgrade-Downgrade-Guide/g-pre-upgrade-checklists.html)**  
   Cisco page updated **July 12, 2026**. Use this checklist together with this framework before an APIC software upgrade; this script does not replace Cisco's pre-upgrade validation process.

Cisco explicitly directs operators to check the **APIC Release Notes for the release being used**. The applicable Release Notes, Cisco support/upgrade matrix, Cisco Recommended Releases guidance, and APIC compatibility catalog remain authoritative for the supported CIMC/HUU version and permitted upgrade path. If this repository conflicts with current Cisco documentation for the release being upgraded, follow the current Cisco documentation and stop the automation until the discrepancy is reviewed.

## How the Cisco Guidance Maps to This Script

The framework enforces or assists the following documented requirements:

- Determine the APIC PID/UCS platform before selecting HUU firmware.
- Validate the CIMC/HUU release against the APIC software release.
- Validate the Cisco-published **MD5 checksum** before the maintenance action.
  SHA-256 and exact file size can also be configured as additional controls.
- Confirm APIC/CIMC reachability and APIC cluster health before disruption.
- Upgrade only one CIMC at a time in the operator-assisted legacy workflow.
- For legacy HUU: remind the engineer to set a BIOS Administrator password,
  use vKVM/virtual media, warm boot, F6, boot vDVD, choose **Update All**, and
  **not enable Cisco IMC Secure Boot**.
- For ACI 6.2(1)+ APIC-managed CIMC: remind the engineer to use the integrated
  Controller & CIMC workflow, validate each APIC/CIMC, clear Catastrophic
  validation failures, submit, and wait for **Completed**.
- Collect detailed Redfish firmware inventory before and after HUU so changes
  beyond CIMC/BIOS are visible in the report.
- Wait for CIMC HTTPS, APIC HTTPS/REST API availability, APIC command
  readiness, and fully-fit cluster health before proceeding to the next
  controller.

### Important ACI 6.2(1)+ CIMC rule

The APIC-integrated CIMC workflow supports only HUU versions allowed by the APIC
compatibility catalog (`compatRsSuppHw`). Cisco notes that the APIC Release Notes
can list additional supported CIMC versions; when the desired version is not
supported by the integrated workflow, use the CIMC user-interface/HUU method.

## Requirements

- A Linux Ansible control node or Windows Subsystem for Linux
- Python 3.8 or newer on the control node
- `ansible-core` 2.13.x (pinned -- see Python/Ansible compatibility note below)
- HTTPS access from the control node to every CIMC
- HTTPS access from the control node to every APIC (all APIC interaction uses
  the REST API -- no SSH or CLI access to the APIC is required or used)
- Redfish enabled and available on each CIMC
- Approved Cisco HUU firmware and APIC compatibility information
- CIMC and APIC credentials available at run time
- For legacy HUU: working CIMC vKVM/virtual media and a BIOS Administrator
  password as required by Cisco's procedure

### Python / ansible-core compatibility note

This project targets a control node running **Python 3.8**. As of this
writing, every actively maintained `ansible-core` release (2.19, 2.20, 2.21)
requires Python 3.11+ on the control node -- none support Python 3.8. The
newest `ansible-core` release that still supports a Python 3.8 control node
is the **2.13.x** series, which reached end of life in November 2023 and no
longer receives security or bug fixes.

Pinning to `ansible-core==2.13.13` is a deliberate, accepted tradeoff to
support this control node's Python version. Every playbook in this repo has
been syntax-checked and re-verified to run correctly against 2.13.13
specifically (not just a newer version). If/when the control node's Python is
upgraded past 3.8, move to a currently supported `ansible-core` release (see
the [ansible-core support matrix](https://docs.ansible.com/projects/ansible-core/devel/reference_appendices/release_and_maintenance.html))
and drop this pin.

Install Ansible in a virtual environment:

```bash
python3 -m venv .venv
source .venv/bin/activate
python -m pip install --upgrade pip
python -m pip install "ansible-core==2.13.13"
```

**WSL/Ubuntu note:** if `python3 -m venv .venv` fails with a message telling
you to install a `python3.X-venv` package, the `venv` module isn't bundled
with your distro's Python by default. Install it, then retry:

```bash
sudo apt update
sudo apt install python3.14-venv   # match the version number in the error
```

## Initial Setup

### 1. Clone the repository

```bash
git clone https://github.com/<your-github-username>/apic-cimc-ansible.git
cd apic-cimc-ansible
```

### 2. Create the local inventory

```bash
cp inventory.example.yml inventory.yml
```

Edit `inventory.yml`. Every CIMC must include its associated APIC and the
Cisco-approved target CIMC version.

### 3. Credentials

CIMC and APIC credentials are prompted at runtime and are not stored in the
repository. CIMC and APIC almost always use different credentials, so each is
prompted separately; password prompts are masked and credential-bearing tasks
use `no_log: true` where appropriate.

`playbooks/07_operator_assisted_upgrade.yml` only prompts once per credential
set per run -- APIC credentials entered in its first play are shared with the
rest of that playbook's plays automatically, so you won't be asked again
partway through an upgrade.

### 4. Create the compatibility database

```bash
cp data/cimc_compatibility.example.yml data/cimc_compatibility.yml
```

Populate it only with paths approved by the APIC Release Notes / Recommended
Releases guidance for the exact APIC PID, current APIC release, target APIC
release, and HUU version.

### 5. Configure firmware validation

Edit `group_vars/all.yml`:

```yaml
firmware_url: "https://firmware.example.net/path/image.iso"
firmware_expected_md5: "<32-character-Cisco-published-md5>"
firmware_expected_sha256: ""       # optional extra control
firmware_expected_size_bytes: 0    # optional extra control
```

The operator-assisted playbook revalidates the firmware before any disruptive
manual action.

### 6. Run syntax checks

```bash
ansible-playbook --syntax-check playbooks/01_test_cimc_access.yml
ansible-playbook --syntax-check playbooks/07_operator_assisted_upgrade.yml
```

### 7. Test one CIMC first

```bash
ansible-playbook playbooks/01_test_cimc_access.yml --limit apic01_cimc
```

You'll be prompted for the CIMC username and password; the password won't
show on screen as you type it. Do not continue until HTTPS connectivity and
Redfish authentication succeed.

## Cisco Pre-Upgrade Checklist Items

Cisco's July 12, 2026 APIC guide calls for clearing faults (except understood
staging faults), performing an AES-encrypted configuration export, verifying
out-of-band access to all APICs and switches, verifying CIMC access for all
APICs, verifying console access to switches, and reviewing behavioral changes,
open issues, and known issues in the applicable Release Notes.

This project automates CIMC reachability and APIC cluster/recovery checks. The
engineer/change plan must separately confirm the configuration export, switch
console access, Release Notes review, and any required APIC Pre-Upgrade
Validator execution before the APIC software upgrade.

For APIC 6.2(1)+, Cisco's integrated APIC workflow runs built-in validations and
can use external validation rules. A Catastrophic validation failure must be
corrected before continuing.

## Recommended Execution Order

```bash
ansible-playbook playbooks/01_test_cimc_access.yml --limit apic01_cimc
ansible-playbook playbooks/02_collect_cimc_inventory.yml --limit apic01_cimc
ansible-playbook playbooks/03_check_apic_cluster.yml --limit apic01
ansible-playbook playbooks/04_check_compatibility.yml --limit apic01_cimc
ansible-playbook playbooks/05_validate_firmware.yml
ansible-playbook playbooks/06_test_pre_post_diff.yml --limit apic01_cimc
```

After validation, test the complete assisted workflow on one CIMC/APIC pair:

```bash
ansible-playbook playbooks/07_operator_assisted_upgrade.yml \
  --limit apic01_cimc,apic01
```

`07_operator_assisted_upgrade.yml` independently revalidates the approved HUU
image, checks cluster health immediately before the manual action, displays the
Cisco procedure appropriate to `legacy_huu` or `apic_managed`, and pauses. After
the engineer types `COMPLETE`, it waits for recovery, recollects Redfish data,
creates a diff, verifies the target CIMC version, and requires the APIC cluster
to be fully fit before the next controller.

## APIC Software Upgrade Boundary

This release does **not** trigger an APIC software upgrade. After all CIMCs are
at the Cisco-supported release and the fabric is ready, perform the APIC upgrade
using the chapter appropriate to the current APIC version in the **Cisco APIC
Installation and ACI Upgrade and Downgrade Guide**.

For APIC 6.2(1)+, Cisco documents an orchestrated process starting from APIC1:
Admin > Firmware > Controllers > choose **Controller**, select the firmware,
allow built-in/external validations to finish, correct any Catastrophic failure,
click Install, monitor Status/History, and wait until all controllers are on the
target release, the cluster is fully fit, and post-upgrade activities are
complete before proceeding to switch upgrades.

Do not manually reload/decommission APICs or reset the target version if an APIC
upgrade appears stalled; Cisco directs operators to review installer logs,
collect tech-support, and contact TAC when an upgrade does not complete.

## Playbooks

| Playbook | Purpose | Disruptive |
|---|---|---:|
| `01_test_cimc_access.yml` | HTTPS and Redfish validation | No |
| `02_collect_cimc_inventory.yml` | Hardware/component firmware inventory | No |
| `03_check_apic_cluster.yml` | APIC cluster health | No |
| `04_check_compatibility.yml` | Local approved-path validation | No |
| `05_validate_firmware.yml` | URL and Cisco MD5; optional size/SHA-256 | No |
| `06_test_pre_post_diff.yml` | Report-engine test | No |
| `07_operator_assisted_upgrade.yml` | Serial assisted CIMC upgrade and postchecks | Yes, engineer-controlled |
| `site.yml` | Complete CIMC preparation workflow | Yes, engineer-controlled |

## Reports

Reports are written locally under `reports/` and excluded from Git. The before
and after reports include Redfish firmware-inventory member details when the
CIMC exposes them, allowing review of HUU component changes in addition to CIMC
and BIOS versions.

## Important Safety Notes

- Follow the APIC Release Notes and Cisco Recommended Releases document for the
  exact supported HUU/CIMC version.
- Complete the Cisco pre-upgrade checklist before the APIC software upgrade.
- Confirm out-of-band access to all relevant APIC/CIMC systems.
- For legacy HUU, do not enable Cisco IMC Secure Boot when Cisco's procedure
  instructs you to select NO.
- During APIC database conversion/upgrade stages, do not introduce disruptive
  manual actions.
- Stop if the APIC cluster is not fully fit.
- Test this release in a non-production environment before production use.

## Known Limitations

- APIC cluster health is read from the REST API's `infraWiNode` class
  (the same data `acidiag avread` reports). Exact health-string formatting
  can vary by APIC release; the match pattern is configurable through
  `apic_cluster_health_pattern`.
- The inventory collector uses the first Redfish System and Manager resource,
  which matches common standalone CIMC implementations.
- The local compatibility database is manually maintained and is not a
  substitute for Cisco Release Notes, Recommended Releases, or the APIC catalog.
- APIC software installation itself is outside the scope of v1.3.1.

## Versioning

This repository uses semantic versioning. The current release is `1.3.6`.

## License

MIT License. See `LICENSE`.
# apic-cimc-ansible-v1.3.6
