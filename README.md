<h1 align="center">
<img width="460" src="https://raw.githubusercontent.com/cisagov/cool/develop/assets/images/cool_logo.png" alt="COOL logo">
</h1>

# COOL Documentation

Documentation for the **Cloud Optimized Operations Lab (COOL)** — an
open-source, infrastructure-as-code platform for running isolated, ephemeral
security-assessment environments on AWS.

This repository contains no Terraform or application code; it is
documentation only, in two sets aimed at two different readers:

## [`cool-system/`](cool-system/README.md) — the platform

Start here if you want to understand what COOL is or stand up your own
deployment:

- [README](cool-system/README.md) — what COOL is, the problems it solves,
  and how operators experience it.
- [ARCHITECTURE](cool-system/ARCHITECTURE.md) — account topology, network
  design, and identity model.
- [REPOSITORIES](cool-system/REPOSITORIES.md) — the catalogue of
  single-purpose `cisagov/*` repositories that make up the platform.
- [GETTING-STARTED](cool-system/GETTING-STARTED.md) — build a complete COOL
  from an empty AWS account, via a seven-step guide series (workstation →
  management account → Terraform backend → core accounts → AMIs → shared
  services → first assessment environment).

## [`admin-operations/`](admin-operations/README.md) — running it day to day

Start here if you administer an existing COOL deployment. The
[runbook index](admin-operations/README.md) organizes step-by-step procedures
for the routine work: identity and access management (FreeIPA and AWS IAM),
provisioning and destroying assessment environments with Terraform,
certificate management, end-of-assessment data archiving, and first-line
support and troubleshooting.

## Conventions

- **Client platforms:** the COOL has been tested with macOS and Linux client
  machines; the guides note the few places where commands differ between the
  two (package managers, clipboard helpers, single sign-on tooling).
- Values the reader substitutes appear in angle brackets:
  `<environment_name>`, `env<X>`, `<instance_id>`.
- The runbooks use `cool.example.gov` as a **placeholder domain**
  (Guacamole at `guac.env<X>.cool.example.gov`, FreeIPA at
  `ipa0.cool.example.gov`, Kerberos realm `COOL.EXAMPLE.GOV`, and the
  `staging-a.cool.example.gov` equivalents). Substitute the domain your COOL
  deployment actually uses.
