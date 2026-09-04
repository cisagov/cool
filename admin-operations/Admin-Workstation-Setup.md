# Admin Workstation Setup

**Purpose:** One-time administrator workstation setup required before performing
IAM user management, environment provisioning/destruction, or troubleshooting
tasks in the COOL (Production or Staging-a).

The COOL has been tested with both macOS and Linux administrator
workstations. Install the tooling below with your platform's package manager
(Homebrew on macOS; `apt`, `dnf`, or similar on Linux); the runbooks call
out the few other places where the platforms differ.

Some of these steps can (and should) be completed in parallel.

## Access and accounts

1. You are assigned to perform IAM user management and/or environment
   provisioning/destruction in the COOL.
1. You possess an account on your organization's collaboration/document
   platform, requested through the ticketing system.
1. Ensure you have **write access to the COOL environment tracking
   spreadsheet**. If not, request access from the DevSecOps team.
1. Ensure you have read/write access to assessment tickets in the ticketing
   system by submitting a ticket for access to the DevSecOps team ticket
   queue.
1. Create a GitHub account using your government (or corporate) email address
   if you do not already have one.
1. Request that the DevSecOps team grant you access to the four COOL
   configuration repositories. These are **private repositories** — the
   DevSecOps team will provide the clone URLs along with access:
    - `cool-tfvars` — IAM user and group configuration
    - `cool-assessment-tfvars` — per-environment assessment configuration
    - `cool-assessment-terraform` — assessment environment Terraform
    - `cool-users` — FreeIPA user/group seeding

## Clone the repositories

In your working directory (the convention below assumes `~/code/cisagov`),
clone the four repositories using the clone URLs provided by the DevSecOps
team:

```console
cd ~/code/cisagov
git clone <cool-tfvars_clone_url>
git clone <cool-assessment-tfvars_clone_url>
git clone <cool-assessment-terraform_clone_url>
git clone <cool-users_clone_url>
```

**NOTE:** Keep git's default local directory names (matching the repository
names) — later runbook commands rely on paths like
`~/code/cisagov/cool-tfvars`.

## For performing COOL IAM user management

1. In the `cisagov` directory, clone each of the group repositories (named in
   the `cool-tfvars` repository) that you will need to work in. These are
   also private repositories; obtain access and clone URLs from the DevSecOps
   team. To start, clone each of the following:
    - `cool-users-non-admin`
    - `cool-admin-provisioner-iam`
    - `cool-auditor-iam`
    - `cool-certificate-manager-iam`
    - `cool-iam-user-group-manager-iam`

    For example, to clone `cool-users-non-admin`:

    ```console
    cd ~/code/cisagov
    git clone <cool-users-non-admin_clone_url>
    ```

1. For each repository you cloned above, create symlinks to its tfvars and
   tfconfig files:

    ```console
    cd ~/code/cisagov/<name-of-repository>
    ln -snf ~/code/cisagov/cool-tfvars/<groupname>/production.tfvars
    ln -snf ~/code/cisagov/cool-tfvars/<groupname>/production.tfconfig
    ln -snf ~/code/cisagov/cool-tfvars/<groupname>/staging-a.tfvars
    ln -snf ~/code/cisagov/cool-tfvars/<groupname>/staging-a.tfconfig
    ```

## For performing COOL provisioning/destruction using Terraform

1. Go to the `cool-assessment-terraform` directory and create symlinks to the
   assessment tfvars and tfconfig files:

    ```console
    cd ~/code/cisagov/cool-assessment-terraform
    ln -snf ~/code/cisagov/cool-assessment-tfvars/production.tfvars
    ln -snf ~/code/cisagov/cool-assessment-tfvars/production.tfconfig
    ln -snf ~/code/cisagov/cool-assessment-tfvars/staging-a/staging-a.tfvars
    ln -snf ~/code/cisagov/cool-assessment-tfvars/staging-a/staging-a.tfconfig
    ```

## Tooling and credentials

1. You have been granted FreeIPA permissions to create user groups, perform
   Guacamole admin functions, and perform host-based access control (HBAC)
   functions.
1. Install the latest AWS Command Line Interface (CLI) for your platform,
   following the AWS documentation.
1. Install Terraform if you have not already (`brew install terraform` on
   macOS; on Linux, use your distribution's package manager or HashiCorp's
   package repository).
1. You have an AWS IAM user, which can be requested from the DevSecOps team
   via the ticketing system.
1. Using your IAM user, create an AWS access key (and secret key) via the AWS
   console.
1. Contact a member of the DevSecOps team to obtain a template of the AWS
   credentials files for both COOL Production and COOL Staging-a. Your AWS
   credentials file (`~/.aws/credentials`) contains your access key and
   secret key plus the `cool-sharedservices-provisionaccount` and
   `cool-env<X>-provisionaccount` profiles (e.g.
   `cool-env0-provisionaccount`) for the instance where you will be
   provisioning. Using `cisagov/aws-profile-sync` to manage these profiles is
   recommended.

## Final verification

After completing the steps above, notify a member of the DevSecOps team
assigned to perform IAM user management or environment provisioning/deletion
so they can complete the remaining setup with you and test your
configuration.
