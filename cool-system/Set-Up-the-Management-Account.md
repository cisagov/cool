# Phase 2 — Set up the management account #

This phase turns a single empty AWS account into an AWS Organization managed by Control Tower, with the `Static` and `Dynamic` OUs, five Static member accounts, and IAM Identity Center configured for MFA-enforced SSO.  This is the only phase that is primarily click-ops; everything after it is Terraform.

> [!NOTE]
> AWS renamed the top-level Organizations account from **master account** to **management account** and renamed **AWS SSO** to **IAM Identity Center**.  The upstream `cisagov` code still uses `master` in directory and profile names (`cool-accounts/master/`, `cool-master-*`).  This guide uses AWS's current names in prose but keeps the upstream identifiers in commands so they match what you will actually type.

# Prerequisites #

* [Prepare Your Workstation](Prepare-Your-Workstation.md) is complete.
* You have root credentials (with MFA) for a brand-new AWS account that is **not** already a member of an AWS Organization.
* You have at least eight distinct email addresses available for member accounts (plus-addressing is fine).

# Sign in as the root user #

1. Browse to [https://console.aws.amazon.com/](https://console.aws.amazon.com/).
1. Choose **Root user**, enter the root email address for the management account, and complete MFA.

You will only use root for this phase.  After Identity Center is configured you will switch to an SSO user and should not sign in as root again except for [tasks that require it](https://docs.aws.amazon.com/accounts/latest/reference/root-user-tasks.html).

# Activate IAM access to billing information #

This lets IAM roles (not just root) view cost and billing data — needed later if you add financial-auditor users.

1. In the top-right account menu, choose **Account**.
1. Scroll to **IAM user and role access to Billing information** and choose **Edit**.
1. Check **Activate IAM Access** and choose **Update**.

# Set up the Control Tower landing zone #

Control Tower creates the AWS Organization, the `Security` OU with `Audit` and `Log Archive` accounts, enables CloudTrail/Config, and turns on IAM Identity Center.

> [!NOTE]
> Older versions of this guide had you create a temporary IAM admin user and launch a throwaway EC2 instance before Control Tower would start.  Neither step is required any more — Control Tower runs its own pre-launch checks and can be launched directly as the root user.

1. Open the [Control Tower console](https://console.aws.amazon.com/controltower).
1. Choose **Set up landing zone**.
1. **Review pricing and select Regions**
    1. Set **Home Region** to `<region>`.
    1. Set **Region deny setting** to **Enabled**.
    1. Under **Select additional Regions for governance**, add any other regions you plan to build AMIs in or deploy assessment environments to.
    1. Choose **Next**.
1. **Configure organizational units**
    1. Leave **Foundational OU** as `Security`.
    1. Leave **Additional OU** as `Sandbox` (it goes unused, but opting out is more work than ignoring it).
    1. Choose **Next**.
1. **Configure shared accounts**
    1. Under **Log archive account**, choose **Create new account** and enter the email address `<account_email_pattern>` with `<account>` = `logarchive`.  Leave the account name as `Log Archive`.
    1. Under **Audit account**, choose **Create new account** and enter the email address `<account_email_pattern>` with `<account>` = `audit`.  Leave the account name as `Audit`.
    1. Choose **Next**.
1. **Additional configurations**
    1. Under **AWS account access configuration**, keep **AWS Control Tower sets up AWS account access with IAM Identity Center**.
    1. Leave the CloudTrail, log-retention, and KMS defaults unless your organization has specific requirements.
    1. Choose **Next**.
1. **Review and set up landing zone** — check the acknowledgement box and choose **Set up landing zone**.

Landing-zone creation takes 30–60 minutes.  Wait for the green **Landing zone setup complete** banner before continuing.

# Create the Static and Dynamic OUs #

1. Open the [AWS Organizations console](https://console.aws.amazon.com/organizations).
1. Choose **AWS accounts** in the left navigation.
1. Select the **Root** container checkbox.
1. From **Actions › Organizational unit**, choose **Create new**.
1. For **Organizational unit name**, enter `Static` and choose **Create organizational unit**.
1. Repeat with the name `Dynamic`.

# Create the Static member accounts #

You need five accounts under the `Static` OU.  Create them via Organizations (Control Tower Account Factory also works, but Organizations is faster for a batch and the OU is registered with Control Tower in the next section either way).

1. In **AWS Organizations › AWS accounts**, choose **Add an AWS account**.
1. Keep **Create an AWS account** selected.
1. Enter **AWS account name** = `DNS` and **Email address** = `<account_email_pattern>` with `<account>` = `dns`.
1. Leave **IAM role name** as `OrganizationAccountAccessRole`.
1. Choose **Create AWS account**.
1. Repeat for each of:
    | Account name | `<account>` in email |
    |---|---|
    | `Images` | `images` |
    | `Shared Services` | `sharedservices` |
    | `Terraform` | `terraform` |
    | `Users` | `users` |
1. Wait until all five show **✓ Created** (a few minutes each).
1. Check the box next to all five new accounts, then **Actions › AWS account › Move**, choose destination **Static**, and **Move AWS accounts**.

Record the twelve-digit account ID of every account (Management, Audit, Log Archive, DNS, Images, Shared Services, Terraform, Users) in your `<tfvars_repo>` — you will need them repeatedly.

# Register both OUs with Control Tower #

Registering an OU enrolls every account in it (creating the `AWSControlTowerExecution` role and applying baseline guardrails) and auto-enrolls any account moved into it later.

1. Open [Control Tower › Organization](https://console.aws.amazon.com/controltower/home/organization).
1. Select the radio button for **Static**.
1. From **Actions**, choose **Register organizational unit**.
1. Check the acknowledgement box and choose **Register OU**.
1. Wait for the **Registered** state (this can take 10–20 minutes for five accounts).
1. Repeat for **Dynamic** (this is fast — the OU is empty).

# Enable resource sharing within the organization #

The Shared Services Transit Gateway is shared with assessment accounts via AWS RAM; that requires org-level sharing to be on.

1. Open the [Resource Access Manager console](https://console.aws.amazon.com/ram/home).
1. Choose **Settings** in the left navigation.
1. Check **Enable sharing with AWS Organizations** and choose **Save settings**.

# Configure IAM Identity Center #

Control Tower has already enabled Identity Center and created an `AWSControlTowerAdmins` group with the `AWSAdministratorAccess` permission set.  You now customize the portal URL, enforce MFA, and create your own SSO user.

## Customize the access-portal URL ##

1. Open the [IAM Identity Center console](https://console.aws.amazon.com/singlesignon).
1. On the **Dashboard**, find the **Settings summary** panel and choose **Customize** under the AWS access portal URL.
1. Enter your chosen subdomain (this becomes `<sso_url>` = `https://<subdomain>.awsapps.com/start`) and confirm.

## Enforce MFA ##

1. In the left navigation, choose **Settings**.
1. Choose the **Authentication** tab.
1. In **Multi-factor authentication**, choose **Configure**.
1. Set **Prompt users for MFA** to **Every time they sign in (always-on)**.
1. Under **Users can authenticate with these MFA types**, check **Security keys and built-in authenticators** and **Authenticator apps**.
1. Under **If a user does not yet have a registered MFA device**, select **Require them to register an MFA device at sign in**.
1. Choose **Save changes**.

## Create your SSO user ##

1. In the left navigation, choose **Users**, then **Add user**.
1. Fill in **Username** (recommended: `<first.last>`), **Email address**, **First name**, **Last name**.
1. Keep **Send an email to this user with password setup instructions** selected.
1. Choose **Next**.
1. On the **Groups** page, check **AWSControlTowerAdmins**.
1. Choose **Next**, review, and **Add user**.
1. Follow the invitation email to set your password and register an MFA device.
1. Optionally delete the auto-created Control Tower SSO user (the one with the management account's root email as username) once your own user works.

## Assign the admins group to every account ##

Control Tower auto-assigns `AWSControlTowerAdmins` to the Management, Audit, and Log Archive accounts.  Extend that to the Static accounts:

1. Under **Multi-account permissions › AWS accounts**, expand the **Static** OU.
1. Check all five Static accounts.
1. Choose **Assign users or groups**.
1. On the **Groups** tab, check **AWSControlTowerAdmins** and choose **Next**.
1. Check **AWSAdministratorAccess**, **AWSOrganizationsFullAccess**, **AWSPowerUserAccess**, and **AWSReadOnlyAccess**, then **Next**.
1. Choose **Submit** and wait for completion.

# Grant Account Factory portfolio access #

Later phases use [`provision-aws-account`](https://github.com/cisagov/provision-aws-account) to create `env<N>` accounts through the Control Tower Account Factory, which is exposed as a Service Catalog product.  The role that Terraform assumes needs access to that portfolio.

1. Open the [Service Catalog console](https://console.aws.amazon.com/servicecatalog).
1. Under **Administration**, choose **Portfolios**.
1. Choose **AWS Control Tower Account Factory Portfolio**.
1. Choose the **Access** tab, then **Grant access**.
1. For **Access type**, keep **IAM Principal**.
1. Choose the **Roles** tab, search for `ControlTowerAdmin`, and check the role named exactly `ControlTowerAdmin` (not the `AWSControlTowerAdmin` service role).
    > [!NOTE]
    > This role does not exist yet — it is created when you bootstrap the management account in [Bootstrap the Core Accounts](Bootstrap-the-Core-Accounts.md).  If it is missing now, skip this step and return to it after that bootstrap; there is a reminder on that page.
1. Choose **Grant access**.

# Sign out of root and sign in via SSO #

1. Sign out of the console.
1. Browse to `<sso_url>`, sign in as your new SSO user, and confirm you can open the **Management** account with the **AWSAdministratorAccess** role.
1. On your workstation, fill in `<sso_url>` and `<management_account_id>` in `~/.aws/<workspace>_config`, then run:

    ```console
    aws sso login --sso-session cool-<workspace>
    aws sts get-caller-identity --profile cool-master-sso-admin
    ```

    The second command should print an ARN containing `AWSReservedSSO_AWSAdministratorAccess` and the management account ID.

# Verify #

* [Control Tower dashboard](https://console.aws.amazon.com/controltower/home/dashboard) shows landing-zone status **Ready** and both `Static` and `Dynamic` OUs **Registered**.
* [Organizations › AWS accounts](https://console.aws.amazon.com/organizations/v2/home/accounts) shows eight accounts in the tree in the correct OUs.
* `aws sts get-caller-identity --profile cool-master-sso-admin` succeeds from your workstation.

---

**Next:** [Bootstrap the Terraform Backend](Bootstrap-the-Terraform-Backend.md) · **Up:** [GETTING-STARTED.md](GETTING-STARTED.md)
