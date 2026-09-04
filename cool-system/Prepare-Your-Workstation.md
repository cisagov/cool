# Phase 1 — Prepare your workstation #

Everything in this guide is driven from a single administrator workstation (macOS or Linux).  This page sets that workstation up: tooling, source checkouts, the private tfvars repository, and the AWS credentials layout that every later phase relies on.

# Prerequisites #

* macOS or Linux with a POSIX shell.  (Windows works via WSL2 but is not covered here.)
* Ability to install software as an administrator.
* A GitHub account with an SSH key configured (`ssh -T git@github.com` succeeds).

# Install required tooling #

| Tool | Minimum version | Verify with |
|---|---|---|
| [AWS CLI v2](https://docs.aws.amazon.com/cli/latest/userguide/getting-started-install.html) | 2.22 | `aws --version` |
| [Terraform](https://developer.hashicorp.com/terraform/install) | 1.10 | `terraform version` |
| [Packer](https://developer.hashicorp.com/packer/install) | 1.10 | `packer version` |
| [Ansible](https://docs.ansible.com/ansible/latest/installation_guide/) | 2.16 | `ansible --version` |
| [Python](https://www.python.org/downloads/) | 3.10 | `python3 --version` |
| [Docker](https://docs.docker.com/engine/install/) + Compose plugin | any recent | `docker compose version` |
| [Session Manager plugin](https://docs.aws.amazon.com/systems-manager/latest/userguide/session-manager-working-with-install-plugin.html) | any recent | `session-manager-plugin` |
| Git | 2.23 | `git --version` |
| jq | any recent | `jq --version` |
| OpenSSL | any recent | `openssl version` |
| OpenVPN (client CLI, for generating the TLS-crypt key) | 2.5 | `openvpn --version` |

> [!NOTE]
> Terraform 1.10+ is required so that the S3 backend can use native `use_lockfile` state locking.  The upstream `cisagov` repositories still list a DynamoDB lock table as a prerequisite; that table is no longer needed and this guide does not create one.

Verify the AWS CLI can drive an SSO login (you will use this constantly):

```console
aws configure list
aws sso login --help
```

# Choose your placeholder values #

Decide these now and use them consistently everywhere.  Record them somewhere your team can find them (a page in your private tfvars repo is a good place).

| Placeholder | Meaning | Example |
|---|---|---|
| `<workspace>` | Short name for this COOL deployment; becomes the Terraform workspace name in every root module. | `dev-b` |
| `<cool_domain>` | DNS subdomain delegated to this deployment. | `dev-b.cool.example.org` |
| `<state_bucket>` | Globally unique S3 bucket name for Terraform state (created in phase 3). | `example-cool-dev-b-terraform-state` |
| `<lambda_bucket>` | Globally unique S3 bucket name for Lambda deployment packages (created in phase 3). | `example-cool-dev-b-lambda-artifacts` |
| `<cert_bucket>` | Globally unique S3 bucket name for TLS certificates (created in phase 4). | `example-cool-dev-b-certificates` |
| `<third_party_bucket_prefix>` | Prefix for the third-party-installer bucket in the Images account (workspace name is appended automatically). | `example-cool-third-party` |
| `<findings_bucket>` | Globally unique S3 bucket name for assessment findings (referenced by the Shared Services baseline). | `example-cool-dev-b-findings` |
| `<region>` | Home AWS region. | `us-east-1` |
| `<sso_url>` | IAM Identity Center start URL (you set the subdomain in phase 2). | `https://example-cool.awsapps.com/start` |
| `<account_email_pattern>` | How you will mint per-account email addresses. | `cool-dev-b+<account>@example.org` |

# Clone the source repositories #

Create a workspace directory and clone the public repositories side by side.  The exact path does not matter, but this guide uses `~/cool/src`:

```console
mkdir -p ~/cool/src
cd ~/cool/src

git clone git@github.com:cisagov/cool-accounts.git
git clone git@github.com:cisagov/provision-aws-account.git
git clone git@github.com:cisagov/cool-configure-aws-account.git
git clone git@github.com:cisagov/cool-images-parameterstore.git
git clone git@github.com:cisagov/cool-dns-certboto.git
git clone git@github.com:cisagov/cool-dns-cyber.dhs.gov.git   # fork/rename for your own <cool_domain>
git clone git@github.com:cisagov/cool-sharedservices-networking.git
git clone git@github.com:cisagov/cool-sharedservices-freeipa.git
git clone git@github.com:cisagov/cool-sharedservices-openvpn.git
git clone git@github.com:cisagov/cool-assessment-terraform.git
git clone git@github.com:cisagov/certboto-docker.git
git clone git@github.com:cisagov/aws-profile-sync.git
git clone git@github.com:cisagov/disable-inactive-iam-users-lambda.git

git clone git@github.com:cisagov/freeipa-server-packer.git
git clone git@github.com:cisagov/openvpn-packer.git
git clone git@github.com:cisagov/guacamole-packer.git
git clone git@github.com:cisagov/samba-packer.git
```

Resulting layout:

```text
~/cool/src/
├── aws-profile-sync/
├── certboto-docker/
├── cool-accounts/
│   ├── audit/
│   ├── dns/
│   ├── dynamic/
│   ├── images/
│   ├── log-archive/
│   ├── master/
│   ├── shared-services/
│   ├── terraform/
│   └── users/
├── cool-assessment-terraform/
├── cool-configure-aws-account/
├── cool-dns-certboto/
├── cool-dns-cyber.dhs.gov/          ← fork this for your own domain
├── cool-images-parameterstore/
├── cool-sharedservices-freeipa/
├── cool-sharedservices-networking/
├── cool-sharedservices-openvpn/
├── disable-inactive-iam-users-lambda/
├── freeipa-server-packer/
├── guacamole-packer/
├── openvpn-packer/
├── provision-aws-account/
└── samba-packer/
```

# Create your private tfvars repository #

Every Terraform root module in this guide is configured with two files that must **never** be committed to a public repo:

* `<workspace>.tfconfig` — backend configuration (which S3 bucket holds state).
* `<workspace>.tfvars` — input variable values (account IDs, CIDR blocks, bucket names, and eventually secrets).

Create one private Git repository to hold all of them, referred to in this guide as `<tfvars_repo>`.  Lay it out to mirror the source tree:

```console
mkdir -p ~/cool/src/<tfvars_repo>
cd ~/cool/src/<tfvars_repo>
git init

mkdir -p cool-accounts/{terraform,users,master,audit,dns,images,log-archive,shared-services,dynamic}
mkdir -p provision-aws-account cool-configure-aws-account
mkdir -p cool-images-parameterstore cool-dns-certboto cool-dns-public
mkdir -p cool-sharedservices-networking cool-sharedservices-freeipa cool-sharedservices-openvpn
mkdir -p cool-assessment-terraform
```

Every `<module>/<workspace>.tfconfig` file has the same one-line content — create it once and copy it everywhere:

```console
cat > backend.tfconfig <<'EOF'
bucket = "<state_bucket>"
EOF

find . -type d -mindepth 1 -not -path './.git*' \
  -exec cp backend.tfconfig {}/<workspace>.tfconfig \;
rm backend.tfconfig
```

The `*.tfvars` files are created per module as you reach each phase.

# Prepare the AWS credentials layout #

The COOL Terraform code assumes a specific set of named AWS CLI profiles.  There are two kinds:

* **Temporary bootstrap profiles** (`cool-<account>-account-admin`) — short-lived Identity Center credentials pasted in during phases 3–4 and deleted afterwards.  These live in `~/.aws/credentials`.
* **Permanent role-assumption profiles** (`cool-<account>-provisionaccount`, `cool-<account>-startstopssmsession`, etc.) — assume an IAM role in the target account, sourced from your IAM user in the `Users` account.  These live in `~/.aws/config` and are what you use forever after.

Because a single administrator can operate several COOL deployments (production, staging, `dev-a`, `dev-b`, …) that all want the same profile names, keep each deployment's credentials in its own file and point the AWS CLI at it with an environment variable:

```console
mkdir -p ~/.aws
touch ~/.aws/<workspace>_credentials
touch ~/.aws/<workspace>_config

export AWS_SHARED_CREDENTIALS_FILE=~/.aws/<workspace>_credentials
export AWS_CONFIG_FILE=~/.aws/<workspace>_config
```

> [!TIP]
> Add those two `export` lines to a `source`-able shell snippet (e.g. `~/cool/env-<workspace>.sh`) so you can switch between deployments without editing your shell rc file.

Seed `~/.aws/<workspace>_config` with the SSO session block and the two management-account SSO profiles you will need first (fill in `<sso_url>` and the management account ID after phase 2):

```ini
[sso-session cool-<workspace>]
sso_start_url = <sso_url>
sso_region = <region>
sso_registration_scopes = sso:account:access

[profile cool-master-sso-admin]
sso_session = cool-<workspace>
sso_account_id = <management_account_id>
sso_role_name = AWSAdministratorAccess
region = <region>

[profile cool-master-sso-orgreadonly]
sso_session = cool-<workspace>
sso_account_id = <management_account_id>
sso_role_name = AWSReadOnlyAccess
region = <region>
```

The permanent `cool-user` base profile and the many `cool-<account>-provisionaccount` profiles are added in [Bootstrap the Core Accounts](Bootstrap-the-Core-Accounts.md) once the underlying IAM user and roles exist.

## Optional: aws-profile-sync ##

If more than one person will administer this COOL, install [`aws-profile-sync`](https://github.com/cisagov/aws-profile-sync) and store the permanent role-assumption profiles in a shared roles file inside `<tfvars_repo>`.  Each administrator then adds a single magic comment to their credentials file:

```ini
[cool-user]
aws_access_key_id     = <YOUR_ACCESS_KEY_ID>
aws_secret_access_key = <YOUR_SECRET_ACCESS_KEY>

#!profile-sync ssh://git@github.com/<your_org>/<tfvars_repo>.git branch=main filename=aws-profiles/<workspace>-roles -- source_profile=cool-user role_session_name=<first.last>
#!profile-sync-stop
```

Running `aws-profile-sync` expands that block into the full set of `cool-*` profiles.  This is optional for a solo build but strongly recommended for a team.

# Verify #

```console
terraform version        # ≥ 1.10
packer version
aws --version             # ≥ 2.22
docker compose version
session-manager-plugin    # prints a "started" banner and exits
ls ~/cool/src             # all repos present
cat ~/.aws/<workspace>_config
```

---

**Next:** [Set Up the Management Account](Set-Up-the-Management-Account.md) · **Up:** [GETTING-STARTED.md](GETTING-STARTED.md)
