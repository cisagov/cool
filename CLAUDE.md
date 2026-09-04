# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with
code in this repository.

## What this repository is

This is **not a code project** — it is documentation for CISA's Cloud
Optimized Operations Lab (COOL), a platform that hosts ephemeral AWS
environments for cybersecurity assessments. There are no build, lint, or
test commands; edits are pure Markdown.

## Directory layout

Everything flows from the root `README.md`:

- `cool-system/` — platform documentation for a public audience: what COOL
  is (`README.md`), design (`ARCHITECTURE.md`), the repo catalogue
  (`REPOSITORIES.md`), and a `GETTING-STARTED.md` guide series for standing
  up a new deployment (seven sequenced pages, Prepare-Your-Workstation
  through Provision-the-First-Assessment-Environment).
- `admin-operations/` — day-to-day administrator runbooks for operating an
  existing COOL deployment. `admin-operations/README.md` is the index; it
  distinguishes one-time setup, day-to-day runbooks, and reference pages.

When adding a page, link it from the appropriate index
(`admin-operations/README.md` or the root `README.md`) — unlinked pages are
effectively orphaned.

## Conventions

- **Filenames**: `Title-Cased-With-Hyphens.md`.
- **Links**: standard relative Markdown links including the `.md` extension,
  e.g. `[Provisioning Assessment Environments](Provisioning-Assessment-Environments.md)`.
- **Placeholders**: angle-bracket tokens the reader substitutes, e.g.
  `<environment_name>`, `env<X>`, `<instance_id>`, `<domain_name>`.
- **Placeholder domain**: all documentation uses `cool.example.gov` (realm
  `COOL.EXAMPLE.GOV`; staging variants under `staging-a.cool.example.gov`).
  Never introduce a real deployment domain.
- **Ordered steps**: every item numbered `1.` (Markdown auto-increments);
  sub-steps indented 4 spaces.
- **Shell snippets**: fenced with ```` ```console ````.
- **Sanitization**: keep the docs free of real hostnames beyond the
  placeholder domain, instance IDs, internal IP addresses, S3 bucket names,
  email addresses, and named individuals. Refer to supporting teams
  generically as "the DevSecOps team" (or "the DevSecOps team POC") — never
  by internal org-unit names.
- **Private repositories** (`cool-tfvars`, `cool-assessment-tfvars`,
  `cool-users`, the per-group IAM repos): refer to them by bare repo name
  only — never with an org prefix, GitHub URL, or clone URL. Clone commands
  use placeholders (`git clone <cool-tfvars_clone_url>`) with a note that
  the DevSecOps team provides access and URLs. Local working paths
  (`~/code/cisagov/<repo>`) are a directory convention and are fine to keep.
  Public repos (e.g. `cisagov/cool-assessment-terraform`,
  `cisagov/certboto-docker`) may keep full links.
- **Runbook page shape**: a purpose statement and prerequisites up front,
  followed by one or more procedure sections of numbered steps.

## Domain context that recurs across pages

The docs reference a constellation of external `cisagov/*` Terraform repos
rather than local code. The main ones:

- `cisagov/cool-accounts` — bootstraps the core AWS accounts (terraform,
  users, master, audit, dns, images, log-archive, shared-services).
- `cisagov/cool-assessment-terraform` + `cool-assessment-tfvars` (private) —
  provision/destroy individual assessment environments via per-environment
  Terraform workspaces.
- `cool-tfvars` (private) + per-group IAM repos (`cool-users-non-admin`,
  `cool-admin-provisioner-iam`, …; also private) — AWS IAM users and groups
  as code.
- `cool-users` (private) — FreeIPA seeding script for groups and HBAC rules.
- `cisagov/cool-sharedservices-*` — networking, FreeIPA cluster, OpenVPN.
- `cisagov/cool-dns-*` / `cisagov/certboto-docker` — DNS zones and
  certificate issuance.
- `cisagov/aws-profile-sync` — manages the many `cool-*-provisionaccount`
  AWS CLI profiles the runbooks assume.

Recurring infrastructure vocabulary: AWS Control Tower org with **Static**
(DNS, Images, Shared Services, Terraform, Users) and **Dynamic**
(per-assessment `env<X>`) OUs; FreeIPA at
`ipa0[.staging-a].cool.example.gov` for identity, groups, and HBAC;
Guacamole (`guac.env<X>...`) for browser-based remote desktop into
environments; OpenVPN + Kerberos (`kinit -f`) for operator access; SSM
Session Manager (profile pattern `cool-env<X>-startstopssmsession`) for
shell access to instances.

Assessment types referenced in the docs: FAST, PCA, RPT, RTA, RVA. `env0`
is special — it hosts shared resources (e.g. shared Nessus) used by other
assessment environments.
