# Usage Guide — Cisco APIC CIMC Upgrade Framework

This guide accompanies every release of this repository. It explains what the
framework does, what it deliberately does not do, and walks through running
it from a clean checkout to a completed CIMC upgrade. It applies to the
current release in this package; see `CHANGELOG.md` and `VERSION` for
release-specific changes.

## 1. What this framework does (and does not do)

**Does:** validates CIMC/APIC reachability, collects hardware and firmware
inventory over Redfish, checks APIC cluster health, validates a local
compatibility decision (which upgrade method is approved for your exact
APIC/CIMC combination), validates the Cisco-published MD5 checksum of the
firmware image, displays the Cisco-documented manual procedure appropriate
to your ACI release, pauses for the engineer to perform that procedure by
hand, then re-validates health and writes a before/after diff report.

**Does not:** push, activate, or install firmware on any CIMC or APIC. It
never calls an undocumented firmware-activation API. The actual disruptive
action — legacy HUU or the APIC-managed CIMC workflow — is always performed
by a human engineer following Cisco's documented steps, which this framework
displays but does not execute. It also does not perform the APIC *software*
upgrade itself (as opposed to the CIMC firmware) — that is a separate,
manual procedure covered in the README's "APIC Software Upgrade Boundary"
section.

**Blast radius:** one CIMC at a time (`serial: 1` in every playbook that
touches the `cimc` group). Every phase aborts immediately if a required
check fails (`any_errors_fatal: true`), and the operator-assisted workflow
specifically re-checks APIC cluster health immediately before *and* after
each individual CIMC's manual action, refusing to proceed if the cluster
isn't fully fit.

## 2. Who should run this

Engineers who are already comfortable with a Linux/WSL shell, `git`, and
basic Ansible usage (`ansible-playbook`, `--limit`, reading a `PLAY RECAP`).
You do not need to write or edit YAML to run a release as shipped — only to
customize inventory, credentials, and compatibility data for your
environment, which are plain key/value files with worked examples.

## 3. Prerequisites

- A Linux control node or Windows Subsystem for Linux (WSL)
- Python and `ansible-core` matching the versions stated in this release's
  `README.md` — check that file for the exact pin before installing anything
- HTTPS reachability from the control node to every CIMC and every APIC --
  all APIC interaction uses the REST API, so no SSH or CLI access to the
  APIC is required or used anywhere in this framework
- Redfish enabled on each CIMC
- The Cisco-approved HUU firmware image and its published MD5 checksum for
  your exact APIC model and target release
- CIMC and APIC login credentials on hand — you will be prompted for these
  interactively every run; they are never stored in the repository
- For legacy HUU specifically: working CIMC vKVM/virtual media access and a
  BIOS Administrator password set on the server, per Cisco's procedure

## 4. One-time setup, per environment

Do this once per control node (or once per person running it):

1. **Create and activate a virtual environment**, then install the pinned
   `ansible-core` version — see the exact commands in `README.md`, under
   "Requirements." Using a venv keeps this install isolated from anything
   else on the control node.

   If `python3 -m venv .venv` fails asking you to install a
   `python3.X-venv` package, that's a distro packaging quirk (common on
   WSL/Ubuntu) — run the `apt install` command the error suggests, then
   retry. See the WSL/Ubuntu note in `README.md` if you hit this.

2. **Create your real inventory** from the example:

   ```bash
   cp inventory.example.yml inventory.yml
   ```

   Edit `inventory.yml` with your actual CIMC and APIC addresses. Every CIMC
   entry must include `associated_apic` and `apic_target_cimc_version`.
   `inventory.yml` is git-ignored — it never gets committed.

3. **Create your compatibility database** from the example:

   ```bash
   cp data/cimc_compatibility.example.yml data/cimc_compatibility.yml
   ```

   Populate it only with combinations (APIC model, current APIC release,
   target APIC release, supported CIMC versions, upgrade method) that you've
   confirmed against the current APIC Release Notes and Cisco's Recommended
   Releases guidance. This file is also git-ignored.

4. **Configure firmware validation** in `group_vars/all.yml`: set
   `firmware_url` and `firmware_expected_md5` (the 32-character MD5 Cisco
   publishes alongside the image). `firmware_expected_sha256` and
   `firmware_expected_size_bytes` are optional extra checks.

Credentials need no setup — every playbook that touches CIMC or APIC prompts
for username/password at the start of the run, with the password input
masked. `07_operator_assisted_upgrade.yml` only asks once per credential set
per run, even though it has multiple internal phases.

## 5. Before every run

- Confirm you're on the intended branch/release and that `inventory.yml`,
  `data/cimc_compatibility.yml`, and the firmware settings match the
  maintenance window you're actually performing.
- Confirm the APIC cluster is healthy and any known faults are understood,
  per Cisco's pre-upgrade checklist (see `README.md`, "Cisco Pre-Upgrade
  Checklist Items"). This framework checks CIMC reachability and APIC
  cluster health automatically — it does **not** check configuration backups,
  switch console access, or Release Notes review for you.
- Have a second engineer or your change record handy — the operator-assisted
  workflow will pause and wait for you mid-run; know who's driving the
  keyboard during that pause.

## 6. Running it, phase by phase

Run each non-disruptive phase against a single controller first, before
widening scope. Every command below prompts for whatever credentials it
needs; type them when asked.

```bash
ansible-playbook playbooks/01_test_cimc_access.yml --limit apic01_cimc
ansible-playbook playbooks/02_collect_cimc_inventory.yml --limit apic01_cimc
ansible-playbook playbooks/03_check_apic_cluster.yml --limit apic01
ansible-playbook playbooks/04_check_compatibility.yml --limit apic01_cimc
ansible-playbook playbooks/05_validate_firmware.yml
ansible-playbook playbooks/06_test_pre_post_diff.yml --limit apic01_cimc
```

**What "success" looks like:** each run ends with a `PLAY RECAP` line per
host. You want `failed=0` and `unreachable=0`. `changed=` counts are normal
and expected — they mean the playbook wrote a report file or created the
`reports/` directory, not that anything failed.

**Where output goes:** every playbook that writes a report now prints the
exact path before and after writing it (for example, `Report file:
/home/you/apic-cimc-ansible/reports/apic01_cimc-current.json`). The path is
resolved relative to the repository, not your current shell directory, so
trust the printed path over any assumption about where you ran the command
from.

`02`'s report is intentionally a trimmed summary (identity, firmware
version, health, timestamps only). `06` and `07` write the full inventory,
including a before/after diff, because their comparison logic needs it.

## 7. Running the operator-assisted upgrade (`07`)

Once every phase above passes cleanly for a given CIMC/APIC pair, run the
assisted workflow:

```bash
ansible-playbook playbooks/07_operator_assisted_upgrade.yml \
  --limit apic01_cimc,apic01
```

What happens, in order:

1. Checks initial APIC cluster health.
2. Re-validates the firmware image's MD5 (and optional SHA-256/size) fresh,
   even if you already ran `05` earlier — this catches a stale or swapped
   image before anything disruptive happens.
3. Collects pre-upgrade CIMC inventory and saves a "before" report.
4. Validates compatibility again and re-checks APIC cluster health
   *immediately* before pausing — if the cluster isn't fully fit at this
   exact moment, it stops here and does not pause for manual action.
5. **Pauses and prints the Cisco-documented procedure** appropriate to your
   compatibility record's `upgrade_method` — either the legacy HUU steps
   (vKVM, virtual media, warm boot, F6, Update All, "select NO" on Cisco IMC
   Secure Boot) or the ACI 6.2(1)+ APIC-managed CIMC steps (Admin > Firmware
   > Controllers > Controller & CIMC upgrade). **Perform that procedure by
   hand now**, on the actual device, following Cisco's documentation — this
   framework only displays the steps, it does not execute them.
6. Waits for you to type `COMPLETE` at the prompt. Anything else is
   rejected and the assert fails.
7. Waits for CIMC HTTPS and APIC HTTPS to come back, then polls the APIC's
   REST API login endpoint until it actually accepts authenticated requests
   (not just respond to a port check).
8. Re-collects inventory, saves an "after" report, writes a diff report, and
   verifies the CIMC landed on the exact target version with healthy status.
9. Re-checks APIC cluster health one final time before considering this
   controller done.

If you're upgrading multiple CIMC/APIC pairs, repeat this command per pair
— the framework processes one at a time by design. `site.yml` chains
playbooks `01` through `04` plus `07` into a single invocation if you'd
rather not run each phase as a separate command, but the same one-CIMC-at-
a-time, pause-for-confirmation behavior still applies within it.

## 8. Troubleshooting

**`sudo apt install python3.X-venv` prompt when creating the venv** — your
distro doesn't bundle the `venv` module for that Python version by default.
Run the suggested `apt install` command, then retry `python3 -m venv .venv`.

**Redfish/HTTPS authentication failure in `01`** — double check the CIMC
credentials you typed, that Redfish is enabled on that CIMC, and that the
control node can reach it over HTTPS. Do not proceed past this point until
it passes.

**"APIC cluster is not fully fit" assertion failure** — stop. Investigate
and resolve the cluster health issue before re-running. Do not bypass this
check by editing `apic_expected_active_controllers` down just to get past
it, unless that number is genuinely wrong for your cluster size.

**"No unique approved record was found" in compatibility validation** — the
APIC model, current/target APIC version, and target CIMC version you typed
don't match any entry in `data/cimc_compatibility.yml`. Add the correct,
Cisco-approved entry rather than adjusting the script's matching logic.

**Firmware MD5 mismatch** — the downloaded image doesn't match the MD5
Cisco published for that release. Re-download from Cisco directly; do not
proceed with a mismatched image.

**Confirmation text rejected in the pause step** — you must type the exact
text shown in the prompt (default `COMPLETE`). It's case-insensitive but
must match exactly otherwise.

## 9. Safety rules (do not bypass these)

- Never edit the framework to call an undocumented firmware-activation API
  or to auto-answer the pause prompt.
- Always follow the currently published Cisco procedure at the pause step,
  even if it has changed since this release was cut — Cisco's live
  documentation is authoritative, not this script.
- One CIMC at a time. Do not widen `--limit` to multiple controllers for the
  disruptive workflow.
- If the APIC cluster upgrade itself (separate from CIMC) appears stalled,
  do not manually reload or decommission an APIC or reset the target
  version — follow Cisco's guidance to check installer logs, collect
  tech-support, and contact TAC.

## 10. Getting help

For anything specific to this framework's behavior, open an issue or PR
comment against this repository. For anything about the Cisco upgrade
procedure itself, defer to the current APIC Release Notes, the Cisco APIC
Installation and ACI Upgrade and Downgrade Guide, and Cisco TAC — this
framework is an aid, not a replacement for Cisco's documentation.
