# Phase 3 — Bootstrap the Terraform backend #

Every Terraform root module in the COOL stores its state in a single S3 bucket in the `Terraform` account.  That bucket does not exist yet, and the Terraform code that creates it wants to store *its own* state in it — the first chicken-and-egg loop.  This page breaks the loop by applying `cool-accounts/terraform` once with local state, then migrating that state into the bucket it just created.

# Prerequisites #

* [Set Up the Management Account](Set-Up-the-Management-Account.md) is complete: the `Terraform` account exists in the `Static` OU, is enrolled in Control Tower, and your SSO user has `AWSAdministratorAccess` on it.
* On your workstation, `AWS_SHARED_CREDENTIALS_FILE` and `AWS_CONFIG_FILE` point at the per-deployment files created in [Prepare Your Workstation](Prepare-Your-Workstation.md).
* You have chosen `<state_bucket>` and `<lambda_bucket>` names.

# Get temporary admin credentials for two accounts #

The `terraform` root has two AWS providers: one targeting the `Terraform` account, and one (`organizationsreadonly`) targeting the management account so it can look up organization metadata.  During bootstrap both must run as short-lived Identity Center admin credentials because the permanent IAM roles do not exist yet.

1. Sign in at `<sso_url>`.
1. For the **Terraform** account, expand it, choose **Command line or programmatic access** next to `AWSAdministratorAccess`, and copy the three exported values into `~/.aws/<workspace>_credentials` under the profile name the code expects:

    ```ini
    [cool-terraform-account-admin]
    aws_access_key_id = <ASIA…>
    aws_secret_access_key = <…>
    aws_session_token = <…>
    ```

1. Repeat for the **Management** account:

    ```ini
    [cool-master-account-admin]
    aws_access_key_id = <ASIA…>
    aws_secret_access_key = <…>
    aws_session_token = <…>
    ```

> [!TIP]
> These session tokens expire after (by default) one hour.  If a `terraform apply` later fails with `ExpiredToken`, just refresh the block from the access portal and rerun.

# Point the code at the temporary profiles #

In `~/cool/src/cool-accounts/terraform`:

1. **Disable the remote backend.**  Open `backend.tf` and comment out **every** line (the S3 bucket does not exist yet, so Terraform must use local state for the first apply):

    ```hcl
    # terraform {
    #   backend "s3" {
    #     …
    #   }
    # }
    ```

1. **Swap the default provider.**  In `providers.tf`, find the `provider "aws"` block with alias-less/default configuration, comment out `profile = "cool-terraform-provisionaccount"`, and uncomment `profile = "cool-terraform-account-admin"` directly below it.
1. **Swap the organizationsreadonly provider.**  In the same file, find the `provider "aws"` block with `alias = "organizationsreadonly"`, comment out `profile = "cool-master-organizationsreadonly"`, and uncomment `profile = "cool-master-account-admin"` directly below it.

Keep track of these three edits — two of them are reverted later on this page and the third in [Bootstrap the Core Accounts](Bootstrap-the-Core-Accounts.md).

# First apply — local state #

1. Create `<tfvars_repo>/cool-accounts/terraform/<workspace>.tfvars`:

    ```hcl
    lambda_bucket_name = "<lambda_bucket>"
    lambda_key         = "disable_inactive_iam_users.zip"
    state_bucket_name  = "<state_bucket>"

    tags = {
      Team        = "<your_team_name>"
      Application = "COOL - Terraform Account"
      Workspace   = "<workspace>"
    }
    ```

1. Initialize with no backend and create the workspace:

    ```console
    cd ~/cool/src/cool-accounts/terraform
    terraform init -upgrade
    terraform workspace new <workspace>
    ```

    > [!NOTE]
    > If you have previously initialized this checkout against a different backend (e.g. for another COOL deployment), add `-reconfigure` to `terraform init`.

1. Apply:

    ```console
    terraform apply -var-file=<tfvars_repo>/cool-accounts/terraform/<workspace>.tfvars
    ```

    Review the plan — it creates (among other things) the `<state_bucket>` and `<lambda_bucket>` S3 buckets, a `ProvisionAccount` IAM role, and CloudWatch alarming — and enter `yes`.

# Migrate state into the new S3 backend #

The bucket now exists; move this root's own state into it.

1. **Restore `backend.tf`** — uncomment everything you commented out earlier.
1. **Swap the backend profile.**  Still in `backend.tf`, comment out `profile = "cool-terraform-backend"` and uncomment `profile = "cool-terraform-account-admin"` directly below it.  (The permanent `cool-terraform-backend` profile is a role assumed via your IAM user in the `Users` account, which does not exist until [Bootstrap the Core Accounts](Bootstrap-the-Core-Accounts.md).)

    > [!IMPORTANT]
    > Leave both `providers.tf` edits (default provider → `cool-terraform-account-admin`, `organizationsreadonly` → `cool-master-account-admin`) in place for now.  Although the `ProvisionAccount` role now exists in this account, its trust policy only allows principals in the `Users` account to assume it — and there are no IAM users there yet.  Both provider edits, plus the `backend.tf` profile swap above, are reverted in [Bootstrap the Core Accounts](Bootstrap-the-Core-Accounts.md) once your permanent IAM user exists.  (The upstream `cool-accounts/terraform/README.md` reverts the default provider one step earlier; in practice that only works if you already have a `Users`-account principal, so this guide defers it.)

1. Ensure `<tfvars_repo>/cool-accounts/terraform/<workspace>.tfconfig` contains:

    ```hcl
    bucket = "<state_bucket>"
    ```

1. Re-initialize against S3.  When prompted **Do you want to migrate all workspaces to "s3"?**, answer `yes`:

    ```console
    terraform init -upgrade \
      -backend-config=<tfvars_repo>/cool-accounts/terraform/<workspace>.tfconfig
    ```

# Upload the disable-inactive-users Lambda #

Every `cool-accounts/*` root deploys a small Lambda that disables IAM users after a period of inactivity.  The deployment package must be in `<lambda_bucket>` before the next apply will succeed.

1. Build the package (see the [`disable-inactive-iam-users-lambda` README](https://github.com/cisagov/disable-inactive-iam-users-lambda) for details; typically):

    ```console
    cd ~/cool/src/disable-inactive-iam-users-lambda
    pip install --requirement requirements.txt --target package
    (cd package && zip -r ../disable_inactive_iam_users.zip .)
    zip -g disable_inactive_iam_users.zip lambda_handler.py
    ```

1. Upload it:

    ```console
    aws s3 cp disable_inactive_iam_users.zip s3://<lambda_bucket>/ \
      --profile cool-terraform-account-admin --region <region>
    ```

# Second apply — remote state #

```console
cd ~/cool/src/cool-accounts/terraform
terraform apply -var-file=<tfvars_repo>/cool-accounts/terraform/<workspace>.tfvars
```

This apply should now complete cleanly.  From this point on, `cool-accounts/terraform` is in its steady-state configuration **except** for three temporary edits that you will unwind in the next phase:

| File | Temporary value | Revert after |
|---|---|---|
| `backend.tf` | `profile = "cool-terraform-account-admin"` | `Users` account is bootstrapped |
| `providers.tf` (default) | `profile = "cool-terraform-account-admin"` | `Users` account is bootstrapped |
| `providers.tf` (`organizationsreadonly`) | `profile = "cool-master-account-admin"` | Management account is bootstrapped |

# Confirm subscription emails #

Each account bootstrap creates an SNS topic that emails the account owner on IAM user/group changes.  You will receive an **AWS Notification – Subscription Confirmation** email at the `Terraform` account address; click **Confirm subscription** within three days.  You will get one of these for every account you bootstrap in the next phase — confirm each one as it arrives.

# Verify #

```console
aws s3 ls s3://<state_bucket>/ --profile cool-terraform-account-admin --region <region>
```

You should see an `env:/` prefix containing `<workspace>/cool-accounts/terraform/terraform.tfstate` (exact key layout depends on the backend `key` and `workspace_key_prefix` in `backend.tf`).

```console
aws iam get-role --role-name ProvisionAccount \
  --profile cool-terraform-account-admin --region <region>
```

Should return the role.

---

**Next:** [Bootstrap the Core Accounts](Bootstrap-the-Core-Accounts.md) · **Up:** [GETTING-STARTED.md](GETTING-STARTED.md)
