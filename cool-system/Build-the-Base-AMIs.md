# Phase 5 — Build the base AMIs #

The Shared Services and assessment-environment Terraform roots launch EC2 instances from custom AMIs that must already exist in the `Images` account.  This phase populates the SSM Parameter Store values those AMIs bake in, creates a per-repo build user, builds each AMI with Packer, and shares it to the accounts that will launch it.

Only four images are strictly required to reach a working `env0` in this guide:

| Image | Repository | Consumed by | Shared to |
|---|---|---|---|
| FreeIPA server | [`cisagov/freeipa-server-packer`](https://github.com/cisagov/freeipa-server-packer) | `cool-sharedservices-freeipa` | Shared Services |
| OpenVPN gateway | [`cisagov/openvpn-packer`](https://github.com/cisagov/openvpn-packer) | `cool-sharedservices-openvpn` | Shared Services |
| Guacamole | [`cisagov/guacamole-packer`](https://github.com/cisagov/guacamole-packer) | `cool-assessment-terraform` | every `env<N>` |
| Samba | [`cisagov/samba-packer`](https://github.com/cisagov/samba-packer) | `cool-assessment-terraform` | every `env<N>` |

The assessment tooling images (`kali-packer`, `nessus-packer`, `teamserver-packer`, `egress-assess-packer`, etc.) follow the same pattern; build them later as needed for the assessment types you plan to run.

# Prerequisites #

* [Bootstrap the Core Accounts](Bootstrap-the-Core-Accounts.md) is complete.  In particular the `Images` account baseline created:
    * A KMS key aliased `alias/cool-amis` for AMI encryption.
    * A VPC and public subnet both tagged `Name = "AMI Build"` for Packer to launch build instances into.
    * The `EC2AMICreate` and `ProvisionEC2AMICreateRoles` IAM roles.
* `cool-images-parameterstore` has been applied and the `cool-images-parameterstorefullaccess` profile works.
* Packer, Ansible, and Python are installed per [Prepare Your Workstation](Prepare-Your-Workstation.md).

# Populate SSM Parameter Store #

Several images read secrets from Parameter Store in the `Images` account at build time.  Create them now.  Use strong random values; you will not need to type most of these again.

```console
export AWS_PROFILE=cool-images-parameterstorefullaccess
export AWS_REGION=<region>

put () { aws ssm put-parameter --type SecureString --name "$1" --value "$2" --overwrite; }

put /freeipa/admin_password              "$(openssl rand -base64 24)"
put /freeipa/directory_service_password  "$(openssl rand -base64 24)"
put /freeipa/realm                        "$(echo <cool_domain> | tr '[:lower:]' '[:upper:]')"

put /guacamole/postgres_username          guacamole
put /guacamole/postgres_password          "$(openssl rand -base64 24)"
put /rdp/username                         guacuser
put /rdp/password                         "$(openssl rand -base64 24)"
put /vnc/username                         guacuser
put /vnc/password                         "$(openssl rand -base64 24)"
put /vnc/sftp/windows_base_directory      'C:\Users'

ssh-keygen -t ed25519 -N '' -f /tmp/vnc_sftp_key
put /vnc/ssh/ed25519_private_key          "$(cat /tmp/vnc_sftp_key)"
put /vnc/ssh/ed25519_public_key           "$(cat /tmp/vnc_sftp_key.pub)"
rm /tmp/vnc_sftp_key /tmp/vnc_sftp_key.pub

put /openvpn/server/tlscrypt.key          "$(openvpn --genkey secret /dev/stdout)"
put /openvpn/server/dh4096.pem            "$(openssl dhparam 4096 2>/dev/null)"

unset AWS_PROFILE AWS_REGION
```

> [!NOTE]
> `openssl dhparam 4096` can take several minutes.
>
> The FreeIPA and OpenVPN images also look for CDM/Nessus/CrowdStrike parameters (`/cdm/nessus_hostname`, `/cdm/falcon/customer_id`, …).  If your organization does not use those agents, build with `-var cdm_enabled=false` (the default) and the parameters are not required.
>
> If a specific `*-packer` build fails on a missing parameter not listed above, the error message names the exact key — create it and rebuild.  The set of required parameters occasionally grows upstream.

# Per-image build procedure #

Each `*-packer` repository is laid out identically.  Repeat the steps below for each of `freeipa-server-packer`, `openvpn-packer`, `guacamole-packer`, `samba-packer` (and any tooling images you need).  In the commands, `<repo>` is the repository directory name.

## Create the build user ##

Each repo's `terraform-build-user/` directory creates an IAM user in the `Images` account with exactly the permissions Packer needs, plus a per-repo `EC2AMICreate-build-<repo>` role.

```console
cd ~/cool/src/<repo>/terraform-build-user
```

Create `<tfvars_repo>/<repo>/terraform-build-user/<workspace>.tfvars`:

```hcl
terraform_state_bucket = "<state_bucket>"
```

```console
terraform init -upgrade \
  -backend-config=<tfvars_repo>/<repo>/terraform-build-user/<workspace>.tfconfig
terraform workspace new <workspace>
terraform apply -var-file=<tfvars_repo>/<repo>/terraform-build-user/<workspace>.tfvars
```

The apply outputs an access key ID and secret for `build-<repo>`.  Add two profiles:

```ini
[build-<repo>]
aws_access_key_id     = <AKIA…>
aws_secret_access_key = <…>

[profile cool-images-ec2amicreate-<repo>]
role_arn = arn:aws:iam::<images_account_id>:role/EC2AMICreate-build-<repo>
source_profile = build-<repo>
role_session_name = <first.last>
region = <region>
```

The `[build-<repo>]` block goes in `~/.aws/<workspace>_credentials`; the `[profile …]` block in `~/.aws/<workspace>_config`.

## Build the image ##

```console
cd ~/cool/src/<repo>

pip install --requirement requirements-dev.txt
ansible-galaxy role install --force --force-with-deps \
  --role-file ansible/requirements.yml
ansible-galaxy collection install --force --force-with-deps \
  --requirements-file ansible/requirements.yml
packer init .

AWS_PROFILE=cool-images-ec2amicreate-<repo> \
  packer build --timestamp-ui \
    -var release_tag="$(./bump-version show)" \
    -var is_prerelease=true \
    .
```

Notes:

* `freeipa-server-packer` and `openvpn-packer` accept `-var cdm_enabled=false` (default) to skip the CDM/Nessus/CrowdStrike agents entirely.  If you *do* enable CDM you must also pass `-var build_bucket=<third_party_bucket_prefix>-<workspace>` and pre-stage the vendor installers there.
* Builds run a temporary EC2 instance in the `Images` account and take 10–30 minutes each.  You can run several in parallel from separate terminals.
* On success Packer prints the new AMI ID; it is also visible in the `Images` account EC2 console under **AMIs**.

## Share the image to consuming accounts ##

Each repo's `terraform-post-packer/` directory grants `LaunchPermission` on the newest AMI(s) to the accounts whose names match a regex (`^Shared Services` for FreeIPA/OpenVPN, `^env[[:digit:]]+$` for Guacamole/Samba).

```console
cd ~/cool/src/<repo>/terraform-post-packer
terraform init -upgrade \
  -backend-config=<tfvars_repo>/<repo>/terraform-post-packer/<workspace>.tfconfig
terraform workspace new <workspace>
terraform apply
```

> [!IMPORTANT]
> For Guacamole and Samba, `terraform apply` here is a **no-op** right now because there are no `env<N>` accounts yet.  You will re-run it in [Provision the First Assessment Environment](Provision-the-First-Assessment-Environment.md) after `env0` exists.  Running it now is harmless and creates the workspace.

# Verify #

```console
aws ec2 describe-images --owners self \
  --profile cool-images-provisionaccount --region <region> \
  --query 'Images[].[Name,ImageId,CreationDate]' --output table
```

You should see at least one AMI per repo you built.  For FreeIPA and OpenVPN, confirm the Shared Services account can see them:

```console
aws ec2 describe-images \
  --owners <images_account_id> \
  --profile cool-sharedservices-provisionaccount --region <region> \
  --query 'Images[].[Name,ImageId]' --output table
```

---

**Next:** [Deploy Shared Services](Deploy-Shared-Services.md) · **Up:** [GETTING-STARTED.md](GETTING-STARTED.md)
