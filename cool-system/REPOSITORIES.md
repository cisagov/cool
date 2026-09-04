# COOL repository catalogue #

COOL is intentionally split across many small, single-purpose
repositories.  Each is independently versioned, reviewed, and
`terraform apply`-ed.  This page is the map.

Two kinds of Terraform repository appear below:

- **Root config** — a top-level Terraform configuration with its own
  backend and workspace(s).  You `cd` into it and run
  `terraform apply`.  These are the things you actually deploy.
- **Module** — reusable Terraform consumed by a root config via
  `module "…" { source = "github.com/cisagov/…" }`.  You generally
  don't apply these directly.

Repositories marked *(archived)* are kept for reference only.

---

## AWS account foundation ##

Creates and baselines the AWS Organization, Control Tower landing
zone, and the per-account IAM roles that everything else assumes.

| Repository | Type | Purpose |
| --- | --- | --- |
| [`cool-accounts`](https://github.com/cisagov/cool-accounts) | Root config (one subdir per account) | Bootstraps every core account — Terraform state bucket/lock table, Users-account IAM users, base `ProvisionAccount` / `StartStopSSMSession` roles in each account. **Start here.** |
| [`provision-aws-account`](https://github.com/cisagov/provision-aws-account) | Root config | Mints new AWS accounts via Control Tower Account Factory (used to create each `env<N>` Dynamic account). |
| [`cool-configure-aws-account`](https://github.com/cisagov/cool-configure-aws-account) | Root config | Post-creation finishing for a new account: SSO permission-set assignments and service-quota increase requests. |
| [`cool-master-cur`](https://github.com/cisagov/cool-master-cur) | Root config | Replicates AWS Cost & Usage Report data out of the master account for billing analysis. |

---

## Shared services (hub) ##

Long-lived infrastructure in the **Shared Services** account that
every assessment environment depends on.

| Repository | Type | Purpose |
| --- | --- | --- |
| [`cool-sharedservices-networking`](https://github.com/cisagov/cool-sharedservices-networking) | Root config | Shared Services VPC, subnets, NAT, VPC endpoints, private Route 53 zone, and the **Transit Gateway hub** shared with all spoke accounts. |
| [`cool-sharedservices-freeipa`](https://github.com/cisagov/cool-sharedservices-freeipa) | Root config | Three-node FreeIPA cluster (directory, Kerberos KDC, HBAC) — the operator identity plane. |
| [`cool-sharedservices-openvpn`](https://github.com/cisagov/cool-sharedservices-openvpn) | Root config | OpenVPN gateway, FreeIPA-authenticated and MFA-enforced — the single network entry point for operators. |
| [`cool-sharedservices-cdm`](https://github.com/cisagov/cool-sharedservices-cdm) | Root config | *CISA-specific.* Site-to-site VPN and log forwarding to CISA's CDM monitoring. Skip unless you have an equivalent. |
| [`cool-sharedservices-nessus`](https://github.com/cisagov/cool-sharedservices-nessus) *(archived)* | Root config | Shared Nessus scanner (superseded by per-environment Nessus instances). |

---

## DNS & certificates ##

Live in the **DNS** account.

| Repository | Type | Purpose |
| --- | --- | --- |
| [`cool-dns-certboto`](https://github.com/cisagov/cool-dns-certboto) | Root config | S3 certificate bucket + Route 53 IAM permissions that `certboto-docker` uses to issue Let's Encrypt certs via DNS-01. |
| [`cool-dns-cyber.dhs.gov`](https://github.com/cisagov/cool-dns-cyber.dhs.gov) | Root config | *CISA example.* Public Route 53 hosted zone. **Fork and rename** for your own domain. |
| [`cool-dns-57.69.64.in-addr.arpa`](https://github.com/cisagov/cool-dns-57.69.64.in-addr.arpa) | Root config | *CISA example.* Reverse (PTR) zone. **Fork** if you own a delegated reverse zone. |
| [`cool-dns-dmarc-import`](https://github.com/cisagov/cool-dns-dmarc-import) | Root config | *CISA example.* Resources for DMARC aggregate-report ingestion (`cisagov/dmarc-import`). Optional. |

---

## Images & Parameter Store ##

Live in the **Images** account.

| Repository | Type | Purpose |
| --- | --- | --- |
| [`cool-images-parameterstore`](https://github.com/cisagov/cool-images-parameterstore) | Root config | IAM roles for reading/writing the SSM Parameter Store secrets that AMIs and instances consume (FreeIPA passwords, OpenVPN keys, etc.). |
| [`cool-images-assessment-images`](https://github.com/cisagov/cool-images-assessment-images) | Root config | S3 storage and access roles for assessment-produced disk images / artifacts. |
| [`cool-images-vmimport`](https://github.com/cisagov/cool-images-vmimport) | Root config | The AWS `vmimport` service role, required to import VM images as AMIs. |
| [`cool-windows-ami-sharing`](https://github.com/cisagov/cool-windows-ami-sharing) | Root config | Shares Windows AMI launch permissions with every Dynamic account. |

---

## IAM groups & cross-account roles ##

These create IAM **groups** in the **Users** account and the
matching assumable **roles** in target accounts, so that humans get
exactly the cloud-side permissions their job needs and nothing more.

| Repository | Purpose |
| --- | --- |
| [`cool-users-non-admin`](https://github.com/cisagov/cool-users-non-admin) | The IAM users themselves (everyone who isn't a break-glass admin) and their group memberships. |
| [`cool-admin-provisioner-iam`](https://github.com/cisagov/cool-admin-provisioner-iam) | Permissions for admins who apply *all* COOL Terraform (core + assessments). |
| [`cool-assessment-provisioner-iam`](https://github.com/cisagov/cool-assessment-provisioner-iam) | Permissions for staff who only provision/destroy assessment environments. |
| [`cool-assessment-images-manager-iam`](https://github.com/cisagov/cool-assessment-images-manager-iam) | Permissions to manage assessment image artifacts. |
| [`cool-certificate-manager-iam`](https://github.com/cisagov/cool-certificate-manager-iam) | Permissions to run `certboto-docker` and write to the cert bucket. |
| [`cool-iam-user-group-manager-iam`](https://github.com/cisagov/cool-iam-user-group-manager-iam) | Permissions to manage IAM users/groups in the Users account. |
| [`cool-auditor-iam`](https://github.com/cisagov/cool-auditor-iam) | Read-only auditor access across all accounts. |
| [`cool-ses-send-email-iam`](https://github.com/cisagov/cool-ses-send-email-iam) | Groups allowed to send email via SES and manage the SES suppression list. |
| [`cool-was-db-iam`](https://github.com/cisagov/cool-was-db-iam) | Read/write access to the Web Application Scanning DB (User Services). |
| [`cool-cdm-cloudtrail-tf-module`](https://github.com/cisagov/cool-cdm-cloudtrail-tf-module) | Module — *CISA-specific.* Lets CDM read an account's CloudTrail. |

---

## Assessment environments (spokes) ##

| Repository | Type | Purpose |
| --- | --- | --- |
| [`cool-assessment-terraform`](https://github.com/cisagov/cool-assessment-terraform) | Root config (one workspace per env) | **The vending machine.** Creates an assessment VPC, TGW attachment, Guacamole gateway, EFS share, and a tfvars-driven count of operator instances (Kali, teamserver, Gophish, Nessus, Windows, …). |
| `cool-assessment-tfvars` *(private — create your own)* | Data | Per-environment `.tfvars` files consumed by the repo above. |
| `cool-tfvars` *(private — create your own)* | Data | Per-workspace `.tfvars` / `.tfconfig` files for the core root configs. |
| `cool-users` *(private — create your own)* | Script | `ipa_initial_seeding.sh` — declarative record of FreeIPA users, groups and HBAC rules. |
| [`guacscanner`](https://github.com/cisagov/guacscanner) | Service | Watches an assessment VPC and auto-registers/deregisters Guacamole connections as EC2 instances appear and disappear. |

---

## Golden images (Packer + Ansible) ##

Each operator-instance type and each shared-service host is launched
from an AMI built in the **Images** account.  Every `*-packer` repo
also contains a `terraform-post-packer/` root config that shares the
freshly built AMI with all Dynamic accounts.

| Repository | AMI for |
| --- | --- |
| [`freeipa-server-packer`](https://github.com/cisagov/freeipa-server-packer) | FreeIPA cluster nodes |
| [`openvpn-packer`](https://github.com/cisagov/openvpn-packer) | OpenVPN gateway |
| [`guacamole-packer`](https://github.com/cisagov/guacamole-packer) | Guacamole remote-desktop gateway |
| [`kali-packer`](https://github.com/cisagov/kali-packer) | Kali Linux operator box |
| [`teamserver-packer`](https://github.com/cisagov/teamserver-packer) | Cobalt Strike teamserver |
| [`pca-gophish-composition-packer`](https://github.com/cisagov/pca-gophish-composition-packer) | Gophish phishing server |
| [`nessus-packer`](https://github.com/cisagov/nessus-packer) | Nessus vulnerability scanner |
| [`egress-assess-packer`](https://github.com/cisagov/egress-assess-packer) | Egress-Assess data-exfil testing tool |
| [`samba-packer`](https://github.com/cisagov/samba-packer) | Samba file server |
| [`terraformer-packer`](https://github.com/cisagov/terraformer-packer) | In-environment Terraform runner (lets operators stand up extra infra mid-engagement) |
| [`debian-packer`](https://github.com/cisagov/debian-packer) / [`ubuntu-server-packer`](https://github.com/cisagov/ubuntu-server-packer) | Generic Linux desktops/servers |
| [`windows-server-packer`](https://github.com/cisagov/windows-server-packer) / [`windows-commando-vm-packer`](https://github.com/cisagov/windows-commando-vm-packer) | Windows Server / Commando VM |
| [`docker-packer`](https://github.com/cisagov/docker-packer) | Base image with Docker pre-installed |
| [`mesa-packer`](https://github.com/cisagov/mesa-packer) | Mesa software-rendering libs (GUI support on GPU-less instances) |
| [`skeleton-packer`](https://github.com/cisagov/skeleton-packer) | Template for adding your own instance type |
| [`ansible-role-xfce-cool`](https://github.com/cisagov/ansible-role-xfce-cool) | Ansible role — XFCE desktop tuned for Guacamole/VNC, used by several images above |

---

## Reusable Terraform modules ##

Pulled in by the root configs above.

| Repository | Purpose |
| --- | --- |
| [`freeipa-server-tf-module`](https://github.com/cisagov/freeipa-server-tf-module) | Stand up a FreeIPA master/replica EC2 instance. |
| [`openvpn-server-tf-module`](https://github.com/cisagov/openvpn-server-tf-module) | Stand up an OpenVPN server instance. |
| [`session-manager-tf-module`](https://github.com/cisagov/session-manager-tf-module) | SSM Session Manager preferences + IAM for an account. |
| [`cert-read-role-tf-module`](https://github.com/cisagov/cert-read-role-tf-module) | IAM role that lets an instance fetch its TLS cert from the cert bucket. |
| [`terraform-state-read-role-tf-module`](https://github.com/cisagov/terraform-state-read-role-tf-module) | IAM role granting read access to specific remote-state paths. |
| [`instance-cw-alarms-tf-module`](https://github.com/cisagov/instance-cw-alarms-tf-module) | Standard CloudWatch alarms for an EC2 instance. |
| [`distributed-subnets-tf-module`](https://github.com/cisagov/distributed-subnets-tf-module) | Spread a set of subnets evenly across AZs. |

---

## Administrator tooling ##

| Repository | Purpose |
| --- | --- |
| [`aws-profile-sync`](https://github.com/cisagov/aws-profile-sync) | Generates and syncs the (large) `~/.aws/credentials` profile set every COOL admin needs from a shared roles file. Strongly recommended. |
| [`certboto-docker`](https://github.com/cisagov/certboto-docker) | Containerized Certbot that issues/renews Let's Encrypt certs via Route 53 DNS-01 and stores them in the cert bucket. |
| [`awssh`](https://github.com/cisagov/awssh) | Convenience wrapper for opening SSM Session Manager shells. |

---

## User-services workload (optional) ##

A separate, non-assessment workload CISA runs alongside COOL.  Listed
for completeness; not required to run assessment environments.

| Repository | Purpose |
| --- | --- |
| [`cool-accounts-userservices`](https://github.com/cisagov/cool-accounts-userservices) | Bootstrap the User Services account. |
| [`cool-userservices-networking`](https://github.com/cisagov/cool-userservices-networking) | VPC + TGW attachment for User Services. |
| [`cool-userservices-dns`](https://github.com/cisagov/cool-userservices-dns) | DNS records for User Services. |
| [`cool-userservices-was-db`](https://github.com/cisagov/cool-userservices-was-db) | Web Application Scanning database table. |
| [`cool-accounts-cyhy-tf-root`](https://github.com/cisagov/cool-accounts-cyhy-tf-root) | Bootstrap Cyber Hygiene (CyHy) accounts. |

---

## Archived / superseded ##

Kept for history; do not deploy.

`cool-accounts-pca`, `cool-pca-iam`, `cool-pca-transitgateway-attachment`,
`cool-accounts-domain-manager`, `cool-domain-manager-iam`,
`cool-domain-manager-networking`, `cool-master-wiz`,
`cool-root-new-user-alarm`, `skeleton-packer-cool`,
`assessor-workbench-packer`, `freeipa-client-packer`,
`freeipa-replica-tf-module`.
