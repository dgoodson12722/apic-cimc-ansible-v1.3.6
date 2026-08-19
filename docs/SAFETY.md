# Safety Model

This project does not initiate CIMC or APIC firmware installation. The disruptive
firmware operation remains an explicit engineer-controlled Cisco procedure.

Version 1.3.1 is aligned to Cisco's documented CIMC workflow by enforcing the
image checksum gate, collecting a baseline, rechecking APIC cluster health
immediately before the manual action, pausing for the engineer, and collecting
post-upgrade Redfish/component inventory before proceeding.

For legacy HUU the engineer remains responsible for vKVM/virtual-media actions,
BIOS password preparation, Update All, selecting NO for Cisco IMC Secure Boot
when prompted, and verifying HUU completion. For ACI 6.2(1)+ APIC-managed CIMC,
the engineer remains responsible for the integrated APIC workflow and its
validation results.

The workflow processes one CIMC at a time and aborts if target-version, health,
or APIC cluster validation fails.
