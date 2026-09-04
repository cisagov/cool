# COOL Administration Runbooks

This directory contains the normalized administrative runbooks for the Cloud
Optimized Operations Lab (COOL). They are written for an administrator who is
familiar with the COOL's purpose and general architecture but does not yet
have day-to-day muscle memory for these procedures.

## How to read these runbooks

- **Placeholders** appear in angle brackets and are substituted by the reader,
  e.g. `<environment_name>`, `env<X>`, `<instance_id>`, `<domain_name>`.
- **COOL instances.** The COOL runs as more than one instance (Production,
  Staging-a, etc.). Where commands differ between instances, both variants are
  shown. The differences are mechanical: Staging-a uses its own AWS
  credentials file (`~/.aws/cool-staging-a-credentials`, selected via the
  `AWS_SHARED_CREDENTIALS_FILE` environment variable), its own Terraform
  backend config and workspaces (`staging-a.tfconfig`, `env<X>-staging-a`),
  and its own DNS/Kerberos namespace (`staging-a.cool.example.gov` /
  `STAGING-A.COOL.EXAMPLE.GOV`).
- **Client platforms.** The COOL has been tested with both macOS and Linux
  client workstations. Most commands are identical on the two; where they
  deviate, the runbooks call it out. The recurring differences: install
  tooling with your platform's package manager (Homebrew on macOS; `apt`,
  `dnf`, or similar on Linux), clipboard helpers (`pbcopy` on macOS,
  `xclip`/`wl-copy` on Linux), and macOS's `app-sso` single sign-on utility,
  which has no Linux counterpart (use `kinit`/`kdestroy` directly).
- **Ticketing and tracking.** The runbooks refer generically to "the
  ticketing system" and "the COOL environment tracking spreadsheet."
  Substitute whatever ticketing system and environment-assignment tracker
  your organization uses.
- **Points of contact.** For everything beyond day-to-day administration —
  platform development, security, and level-2 support — the runbooks refer to
  a single **DevSecOps team** rather than named individuals. In your
  organization these functions may belong to one team or several; map the
  name to whoever fills the role, and obtain the current POC roster from your
  team.

## Initial setup (once per administrator)

Complete this before attempting any of the day-to-day runbooks:

1. [Admin Workstation Setup](Admin-Workstation-Setup.md) — workstation
   configuration, repository access, AWS credentials, FreeIPA permissions.

## Day-to-day administrator runbooks

These are the routine procedures the day-to-day COOL administrators perform,
driven by tickets:

| Runbook | When you use it |
|---|---|
| [FreeIPA User Management](FreeIPA-User-Management.md) | System access requests: add, modify, disable, or delete COOL users. |
| [FreeIPA Group Management](FreeIPA-Group-Management.md) | Assign operators to an assessment; create/remove assessment user groups. |
| [IAM User Management](IAM-User-Management.md) | Create/remove AWS IAM users and group memberships (Terraform-driven). |
| [Provisioning Assessment Environments](Provisioning-Assessment-Environments.md) | Assessment Lab Request tickets: stand up a new assessment environment. |
| [Joining a New Guacamole Host to the FreeIPA Domain](Joining-a-New-Guacamole-Host-to-the-FreeIPA-Domain.md) | Sub-task of provisioning. |
| [Creating or Updating a FreeIPA HBAC Rule](Creating-or-Updating-a-FreeIPA-HBAC-Rule.md) | Sub-task of provisioning. |
| [Verifying the Nessus Setup Script](Verifying-the-Nessus-Setup-Script.md) | Sub-task of provisioning. |
| [Creating and Renewing SSL Certificates](Creating-and-Renewing-SSL-Certificates.md) | Certificate expiry notices for COOL service domains. |
| [Creating SSL Certificates for Assessment Mail Servers](Creating-SSL-Certificates-for-Assessment-Mail-Servers.md) | Before provisioning environments that send email (RPT, PCA, etc.). |
| [Archiving Assessment Data to the Analytic Enclave](Archiving-Assessment-Data-to-the-Analytic-Enclave.md) | "Awaiting Data Archive and Report" tickets at the end of an assessment. |
| [Deleting Assessment Environments](Deleting-Assessment-Environments.md) | Environment destruction tickets after archiving is complete. |

## Keeping these runbooks up to date

Maintaining this documentation is itself part of the administrator's job.
When a procedure changes — a command, a URL, a ticket workflow, a port
baseline, a team responsibility — update the affected runbook as part of
completing the work, not as a someday task. If you hit a step that no longer
matches reality, fix the page (or note the discrepancy at the top of it)
before closing your ticket. Out-of-date runbooks are how single points of
failure form.

## Reference

Consulted as needed rather than followed start-to-finish:

- [Support and Troubleshooting Guide](Support-and-Troubleshooting-Guide.md) —
  symptom-organized guidance for common support tickets.
- The current baseline inbound ports per instance type, used when editing
  Terraform variables files during provisioning, is documented in the private
  `cool-assessment-tfvars` repository.

## Typical assessment lifecycle

For orientation, a full assessment flows through the runbooks in roughly this
order:

1. Mail server certificates created ([mail server certs](Creating-SSL-Certificates-for-Assessment-Mail-Servers.md)), if the assessment sends email.
2. Operators assigned and FreeIPA group created ([group management](FreeIPA-Group-Management.md)).
3. Environment provisioned ([provisioning](Provisioning-Assessment-Environments.md)), including the Guacamole join, HBAC update, and Nessus verification sub-tasks.
4. Assessment runs; support tickets handled via the [troubleshooting guide](Support-and-Troubleshooting-Guide.md).
5. Assessment data archived ([archiving](Archiving-Assessment-Data-to-the-Analytic-Enclave.md)).
6. Environment destroyed and groups removed ([deletion](Deleting-Assessment-Environments.md)).
