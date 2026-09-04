# Phase 7 — Provision the first assessment environment #

This final phase creates one assessment-environment account (`env0`) in the `Dynamic` OU and deploys Guacamole into it.  Reaching a working Guacamole login page proves the whole platform end to end: account vending, cross-account IAM, Transit Gateway routing, AMI sharing, FreeIPA enrollment, DNS, and certificates.

After this, day-2 provisioning of `env1`, `env2`, … follows the ordinary [Provisioning Assessment Environments](../admin-operations/Provisioning-Assessment-Environments.md) runbook, creating each new `env<N>` account the same way the `env0` account is created below.

# Prerequisites #

* [Deploy Shared Services](Deploy-Shared-Services.md) is complete: networking, FreeIPA, and OpenVPN are up and you can `kinit` on the VPN.
* [Build the Base AMIs](Build-the-Base-AMIs.md) produced Guacamole and Samba AMIs (their `terraform-post-packer` step will be re-run below now that a target account exists).
* Your `cool-master-controltoweradmin` and `cool-master-administersso` profiles work (added in [Bootstrap the Core Accounts](Bootstrap-the-Core-Accounts.md)).

# Create the env0 account #

1. Create `<tfvars_repo>/provision-aws-account/<workspace>.tfvars`:

    ```hcl
    accounts = [
      {
        account_email            = "<account_email_pattern with <account>=env0>"
        account_name             = "env0"
        account_org_unit         = "Dynamic"
        provisioned_product_name = "env0-account"
        sso_email                = "<your_sso_email>"
        sso_first_name           = "<First>"
        sso_last_name            = "<Last>"
      },
    ]

    tags = {
      Team        = "<your_team_name>"
      Application = "COOL - Account Provisioning"
      Workspace   = "<workspace>"
    }
    ```

1. Apply.  This drives Control Tower Account Factory via Service Catalog and typically takes 20–30 minutes:

    ```console
    cd ~/cool/src/provision-aws-account
    terraform init -upgrade \
      -backend-config=<tfvars_repo>/provision-aws-account/<workspace>.tfconfig
    terraform workspace new <workspace>
    terraform apply -var-file=<tfvars_repo>/provision-aws-account/<workspace>.tfvars
    ```

    > [!NOTE]
    > If this fails with a Service Catalog access error, revisit the [Grant Account Factory portfolio access](Set-Up-the-Management-Account.md#grant-account-factory-portfolio-access) step — the `ControlTowerAdmin` role must have access to the Account Factory portfolio.

1. Record `<env0_account_id>` from the apply output and add its profiles:

    ```ini
    [profile cool-env0-provisionaccount]
    role_arn = arn:aws:iam::<env0_account_id>:role/ProvisionAccount
    source_profile = cool-user
    role_session_name = <first.last>
    region = <region>

    [profile cool-env0-startstopssmsession]
    role_arn = arn:aws:iam::<env0_account_id>:role/StartStopSSMSession
    source_profile = cool-user
    role_session_name = <first.last>
    region = <region>
    ```

# Assign Identity Center permissions to env0 #

1. Create `<tfvars_repo>/cool-configure-aws-account/env0-<workspace>.tfvars`:

    ```hcl
    account_name_regex       = "^env0$"
    account_quota_profile    = "cool-env0-provisionaccount"
    master_account_workspace = "<workspace>"
    sso_admin_profile        = "cool-master-administersso"
    terraform_state_bucket   = "<state_bucket>"

    tags = {
      Team        = "<your_team_name>"
      Application = "COOL - Configure Account"
      Workspace   = "env0-<workspace>"
    }
    ```

1. Apply:

    ```console
    cd ~/cool/src/cool-configure-aws-account
    terraform init -upgrade \
      -backend-config=<tfvars_repo>/cool-configure-aws-account/<workspace>.tfconfig
    terraform workspace new env0-<workspace>
    terraform apply \
      -var-file=<tfvars_repo>/cool-configure-aws-account/env0-<workspace>.tfvars
    ```

# Bootstrap env0 #

Follow the [general bootstrap pattern](Bootstrap-the-Core-Accounts.md#the-general-per-account-bootstrap-pattern) against `cool-accounts/dynamic/`:

1. Copy `AWSAdministratorAccess` credentials for the **env0** account into `[cool-env0-account-admin]`.
1. In `cool-accounts/dynamic/providers.tf`, swap the default provider from `cool-${var.dynamic_account_name}-provisionaccount` to `cool-${var.dynamic_account_name}-account-admin`.
1. Create `<tfvars_repo>/cool-accounts/dynamic/env0-<workspace>.tfvars`:

    ```hcl
    dynamic_account_name = "env0"
    lambda_bucket_name   = "<lambda_bucket>"
    lambda_key           = "disable_inactive_iam_users.zip"

    tags = {
      Team              = "<your_team_name>"
      Application       = "COOL - env0 Account"
      AssessmentAccount = "true"
      Workspace         = "env0-<workspace>"
    }
    ```

1. Init, workspace, apply, revert, re-apply:

    ```console
    cd ~/cool/src/cool-accounts/dynamic
    terraform init -upgrade \
      -backend-config=<tfvars_repo>/cool-accounts/dynamic/<workspace>.tfconfig
    terraform workspace new env0-<workspace>
    terraform apply -var-file=<tfvars_repo>/cool-accounts/dynamic/env0-<workspace>.tfvars
    # revert providers.tf
    terraform apply -var-file=<tfvars_repo>/cool-accounts/dynamic/env0-<workspace>.tfvars
    ```

# Wire the platform to env0 #

Three platform roots discover assessment accounts dynamically from the AWS Organization; re-applying them now that `env0` exists extends KMS key grants, Transit Gateway sharing, AMI launch permissions, and the findings-bucket role to the new account.

```console
cd ~/cool/src/cool-accounts/images
terraform apply -var-file=<tfvars_repo>/cool-accounts/images/<workspace>.tfvars

cd ~/cool/src/cool-accounts/shared-services
terraform apply -var-file=<tfvars_repo>/cool-accounts/shared-services/<workspace>.tfvars

cd ~/cool/src/cool-sharedservices-networking
terraform apply \
  -var-file=<tfvars_repo>/cool-sharedservices-networking/<workspace>.tfvars \
  -target=aws_ec2_transit_gateway_route.sharedservices_routes \
  -target=aws_ec2_transit_gateway_route_table.tgw_attachments \
  -target=aws_ram_principal_association.tgw

for repo in guacamole-packer samba-packer; do
  (cd ~/cool/src/$repo/terraform-post-packer && terraform apply)
done
```

# Issue the Guacamole certificate #

```console
cd ~/cool/src/certboto-docker
docker compose -f my-docker-compose.yml run certboto certonly -d guac.env0.<cool_domain>
```

# Deploy the assessment environment #

1. Create `<tfvars_repo>/cool-assessment-terraform/env0-<workspace>.tfvars` from the example in the [`cool-assessment-terraform` README](https://github.com/cisagov/cool-assessment-terraform#usage), setting at minimum the account name, `<cool_domain>`, `<state_bucket>`, VPC CIDR (pick a `/16` inside `cool_cidr_block` distinct from Shared Services, e.g. `10.129.0.0/16`), and instance counts.  For a first smoke test set every operations-instance count to `0` — Guacamole is enough to prove the platform.

1. Apply — policy attachment first, then full:

    ```console
    cd ~/cool/src/cool-assessment-terraform
    terraform init -upgrade \
      -backend-config=<tfvars_repo>/cool-assessment-terraform/<workspace>.tfconfig
    terraform workspace new env0-<workspace>

    terraform apply \
      -var-file=<tfvars_repo>/cool-assessment-terraform/env0-<workspace>.tfvars \
      -target=aws_iam_policy.provisionassessment_policy \
      -target=aws_iam_role_policy_attachment.provisionassessment_policy_attachment

    terraform apply \
      -var-file=<tfvars_repo>/cool-assessment-terraform/env0-<workspace>.tfvars
    ```

1. Complete the Transit Gateway attachment from the Shared Services side:

    ```console
    cd ~/cool/src/cool-sharedservices-networking
    terraform apply \
      -var-file=<tfvars_repo>/cool-sharedservices-networking/<workspace>.tfvars
    ```

1. Join the Guacamole instance to the FreeIPA domain — follow [Joining a New Guacamole Host to the FreeIPA Domain](../admin-operations/Joining-a-New-Guacamole-Host-to-the-FreeIPA-Domain.md) using `--profile cool-env0-startstopssmsession`, and run the two Guacamole-specific setup scripts listed there.
1. Create the FreeIPA `env0_<workspace>` group and HBAC rule as described in [FreeIPA Group Management](../admin-operations/FreeIPA-Group-Management.md) and [Creating or Updating a FreeIPA HBAC Rule](../admin-operations/Creating-or-Updating-a-FreeIPA-HBAC-Rule.md), and add your `<first.last>` user to that group.

# Verify #

1. Connect to the OpenVPN gateway.
1. Browse to `https://guac.env0.<cool_domain>/`.
1. Log in with your `<first.last>` FreeIPA credentials.

If the Guacamole login succeeds, the COOL is up.

---

**Up:** [GETTING-STARTED.md](GETTING-STARTED.md) · **Day-2 operations:** [Administrator Runbooks](../admin-operations/README.md)
