# Changelog

## 1.3.6

- Converted all APIC interaction from SSH/CLI (`ansible_connection: ssh` +
  `ansible.builtin.raw: acidiag avread`) to the APIC HTTPS REST API, since
  the control node has HTTPS-only access to the APIC and no SSH/CLI access.
- Rewrote `tasks/check_apic_cluster.yml` to log in via `POST
  /api/aaaLogin.json`, read cluster node health via `GET
  /api/class/infraWiNode.json` (the REST equivalent of `acidiag avread`),
  and log out via `POST /api/aaaLogout.json`. All calls run
  `delegate_to: localhost`, matching the existing Redfish/CIMC pattern.
  The task now accepts optional `target_apic_host`/`target_apic_name` so
  it can check a different APIC's cluster from within a play whose hosts
  are the `cimc` group.
- `playbooks/07_operator_assisted_upgrade.yml`: replaced the port-22/SSH
  recovery wait and `raw` command-readiness check with an HTTPS port-443
  wait followed by polling `POST /api/aaaLogin.json` until the APIC
  accepts authenticated REST requests.
- `inventory.example.yml`: the `apic` group no longer sets
  `ansible_connection: ssh` or SSH credentials; it now uses
  `ansible_connection: local` since every APIC call is HTTP(S) via
  `delegate_to: localhost`, identical to the `cimc` group.
- `group_vars/all.yml`: removed the unused `apic_cluster_command` (`acidiag
  avread`) variable; added `apic_validate_certs` and `apic_request_timeout`
  for the new REST calls.
- Updated `README.md` and `instructions.md` to describe HTTPS-only APIC
  access and remove remaining SSH references.
- No change to CIMC/Redfish behavior, report contents, or credential
  prompting behavior in this release.

## 1.3.5

- Moved the operator run book from `docs/USAGE_GUIDE.md`/`.docx` to
  `instructions.md`/`instructions.docx` at the repository root, as a
  standalone pair of files separate from `README.md`. Content is unchanged.
  No script or playbook behavior changed in this release.

## 1.3.4

- Added `docs/USAGE_GUIDE.md` and a matching `docs/USAGE_GUIDE.docx`: an
  operator-facing run book covering scope/blast-radius, one-time setup,
  phase-by-phase execution, the operator-assisted upgrade walkthrough,
  troubleshooting, and safety rules. Intended to accompany every release.
  No script or playbook behavior changed in this release.

## 1.3.3

- Pinned `ansible-core` to `2.13.13` to support a Python 3.8 control node.
  Every currently maintained ansible-core release (2.19+) requires Python
  3.11+ on the control node; 2.13.x is the newest release that still
  supports Python 3.8, though it has been EOL (no security fixes) since
  November 2023. This tradeoff is documented in the README.
- Re-verified every playbook's `--syntax-check` against ansible-core
  2.13.13 specifically (not just a newer version), and confirmed all
  module options and Jinja filters used in this repo (`stat.checksum_
  algorithm`, `uri.follow_redirects`/`timeout`/`status_code`, `realpath`,
  `regex_findall`, etc.) function correctly under that version.
- Updated README Requirements and venv setup instructions accordingly.

## 1.3.2

- Added a WSL/Ubuntu troubleshooting note to the README for the
  `python3.X-venv` apt package error on `python3 -m venv .venv`.
- `02_collect_cimc_inventory.yml` now writes a trimmed report (address,
  associated_apic, bios_version, cimc_firmware_version, collected_at,
  inventory_hostname, manager_health, manager_model, manufacturer,
  system_health, serial_number) via a new `save_inventory_summary.yml` task;
  `06`/`07` are unaffected and still write the full inventory.
- Removed the unused CIMC firmware-inventory collection chain (Query update
  service, Query firmware inventory collection, Query each firmware
  inventory member, Store detailed firmware inventory members) along with
  the now-pointless `UpdateService` assertion. This drops `collect_cimc_
  inventory.yml` from up to 5+N Redfish calls per CIMC (N = firmware
  component count) down to a fixed 5, and removes the per-item console
  output the old loop produced even under `no_log`.

## 1.3.1

- Aligned the operator workflow to Cisco **Upgrade CIMC on APIC**, updated
  July 3, 2025, and the **Cisco APIC Installation and ACI Upgrade and Downgrade
  Guide**, updated July 12, 2026.
- Added Cisco-published MD5 validation; retained optional SHA-256 and size checks.
- Made firmware validation mandatory inside the operator-assisted workflow.
- Added release-aware manual instructions for legacy HUU and ACI 6.2(1)+
  APIC-managed CIMC upgrades.
- Added immediate APIC cluster validation before each manual CIMC action.
- Added APIC command-readiness polling after reboot.
- Added Redfish collection guards and FirmwareVersion/Version fallback.
- Added detailed Redfish FirmwareInventory member collection for HUU component
  before/after review.
- Added APIC PID/SKU consistency validation when Redfish exposes the SKU.
- Normalized CIMC target-version comparison and engineer confirmation text.
- Corrected README version references and added dated Cisco authoritative
  references and APIC upgrade boundary guidance.

## 1.3.0

- Replaced vault-based CIMC/APIC credential storage with masked runtime
  prompts (`vars_prompt`). Credentials are no longer written to disk in any
  form.
- Removed `group_vars/vault.yml.example` and all `--ask-vault-pass` usage.
- `inventory.example.yml` now sources APIC SSH connection variables from the
  runtime-prompted `apic_username`/`apic_password` instead of vault
  variables.
- `07_operator_assisted_upgrade.yml` broadcasts APIC credentials across its
  plays after the first prompt so the operator is not asked twice in one run.

## 1.2.0

- Replaced placeholder playbooks with functional Ansible workflows.
- Added CIMC HTTPS and Redfish validation.
- Added CIMC system, manager, firmware, and update-service inventory.
- Added APIC cluster health validation through `acidiag avread`.
- Added local compatibility-matrix validation.
- Added firmware URL, size, and SHA-256 validation.
- Added baseline, post-upgrade, and diff JSON reports.
- Added serial operator-assisted upgrade workflow with recovery checks.
- Expanded the README with complete initial setup and execution instructions.
