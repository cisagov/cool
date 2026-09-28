# Sanitizing Assessment Accounts

**Purpose:** To sweep a Dynamic (`env<X>`) AWS account clean of leftover
resources after an assessment environment has been destroyed, before the
account is returned to the pool.

**Prerequisites:** You MUST complete
[Admin Workstation Setup](Admin-Workstation-Setup.md) before proceeding, and
the environment MUST already have been destroyed per
[Deleting Assessment Environments](Deleting-Assessment-Environments.md)
(through the `terraform state list` verification).

## Why this step exists

`terraform destroy` removes only the resources recorded in the Terraform
state. Anything created outside of `cool-assessment-terraform` survives it:

1. Resources created manually through the AWS console or CLI.
1. Resources created by other tooling run inside the account (CloudFormation
   or CDK stacks, terraformer-provisioned infrastructure, etc.).
1. Artifacts created at runtime by instances rather than by Terraform (e.g.
   the `/instance-logs/*` CloudWatch log groups).

Left in place, these leak information between engagements, accrue cost, and
— in the case of half-deleted CloudFormation stacks — can wedge future
cleanup. This runbook uses [aws-nuke](https://github.com/ekristen/aws-nuke)
to enumerate the account's resources and delete everything it recognizes
except a protected baseline (Control Tower, IAM Identity Center, and the
COOL roles the account needs to be re-provisioned).

**NOTE:** aws-nuke does not support every AWS resource type, so a clean run
means "clean within aws-nuke's coverage", not a guarantee that the account
is empty. The DevSecOps team maintains a short list of services known to be
unsupported and worth checking by hand (ask for it with the canonical
config); anything found there is handled manually before closeout.

**WARNING:** aws-nuke is a destructive tool by design. Never point it at any
account other than the Dynamic account you are sanitizing, never run it
without the config file described below, and never skip the dry run. The
config's `blocklist` of protected COOL accounts is a guardrail, not a
substitute for care.

## 1. Install aws-nuke

1. Install the maintained `ekristen/aws-nuke` fork (NOT the archived
   `rebuy-de/aws-nuke`):
    1. macOS: `brew install ekristen/tap/aws-nuke`
    1. Linux: download the release binary for your platform from the
       project's GitHub releases page and place it on your `PATH`.
1. Verify the installation: `aws-nuke version`

## 2. Obtain the configuration file

The DevSecOps team maintains the canonical COOL aws-nuke configuration;
request the current copy rather than authoring your own. It has three jobs:

1. **Blocklist** every COOL account that must never be nuked: the management
   account, Audit, Log Archive, and all Static OU accounts. aws-nuke refuses
   to run against a blocklisted account ID.
1. **Filter (retain)** the baseline resources every Dynamic account must
   keep: everything Control Tower and IAM Identity Center manage, plus the
   COOL roles and account-wide security tooling that the account needs to
   rejoin the pool.
1. **Scope regions** to those COOL uses, plus `global` for IAM.

A starting point, to be reviewed against the current account baseline:

```yaml
regions:
  # "all" is the aws-nuke special value covering every enabled region;
  # "global" covers non-regional services such as IAM. Scanning everything
  # matters because engagements may deploy outside us-east-1 (see the
  # terraformer special case in Deleting Assessment Environments).
  - global
  - all

blocklist:
  # Every non-Dynamic COOL account ID. Obtain the current list from the
  # DevSecOps team.
  - "<management_account_id>"
  - "<audit_account_id>"
  - "<log_archive_account_id>"
  - "<terraform_account_id>"
  - "<users_account_id>"
  - "<dns_account_id>"
  - "<images_account_id>"
  - "<shared_services_account_id>"

presets:
  # Landing-zone baseline: Control Tower and IAM Identity Center.
  landing-zone:
    filters:
      CloudFormationStack:
        - type: glob
          value: "StackSet-AWSControlTower*"
      CloudTrailTrail:
        - type: glob
          value: "aws-controltower*"
      CloudWatchLogsLogGroup:
        - type: glob
          value: "*aws-controltower*"
        - type: glob
          value: "*AWSControlTower*"
      ConfigServiceConfigurationRecorder:
        - type: glob
          value: "aws-controltower*"
      ConfigServiceDeliveryChannel:
        - type: glob
          value: "aws-controltower*"
      EC2DHCPOption:
        - type: glob
          property: tag:Name
          value: "aws-controltower-*"
      EC2InternetGateway:
        - type: glob
          property: tag:Name
          value: "aws-controltower-*"
      EC2RouteTable:
        - type: glob
          property: tag:Name
          value: "aws-controltower-*"
      EC2Subnet:
        - type: glob
          property: tag:Name
          value: "aws-controltower-*"
      EC2VPC:
        - type: exact
          property: tag:Name
          value: "aws-controltower-VPC"
      IAMRole:
        - type: glob
          value: "AWSReservedSSO_*"
        - type: glob
          value: "aws-controltower-*"
        - "AWSControlTowerExecution"
      IAMRolePolicy:
        - type: glob
          property: role:RoleName
          value: "AWSReservedSSO_*"
        - type: glob
          property: role:RoleName
          value: "aws-controltower-*"
      IAMRolePolicyAttachment:
        - type: glob
          property: RoleName
          value: "AWSReservedSSO_*"
        - type: glob
          property: RoleName
          value: "aws-controltower-*"
        - type: exact
          property: RoleName
          value: "AWSControlTowerExecution"
      # SAML providers are identified by their full ARN, so the pattern
      # must match the whole ARN, not just the provider name.
      IAMSAMLProvider:
        - type: glob
          value: "arn:aws:iam::*:saml-provider/AWSSSO_*"
      LambdaFunction:
        - "aws-controltower-NotificationForwarder"
      SNSTopic:
        - type: glob
          value: "aws-controltower-*"

  # COOL baseline: what the Dynamic account needs to be re-provisioned.
  # Keep this aligned with what the cool-*-iam repos and account-wide
  # security tooling actually create in Dynamic accounts.
  cool-baseline:
    filters:
      IAMRole:
        - "ProvisionAccount"
        - "EC2ReadOnly"
        - type: glob
          value: "*SSMSession*"
        - "<security_scanner_role>"
      # Retaining a role is not enough: its inline policies and managed-
      # policy attachments are separate aws-nuke resources and must be
      # filtered too, or the sweep strips the role's permissions.
      IAMRolePolicy:
        - type: glob
          property: role:RoleName
          value: "*SSMSession*"
      IAMRolePolicyAttachment:
        - type: exact
          property: RoleName
          value: "ProvisionAccount"
        - type: exact
          property: RoleName
          value: "EC2ReadOnly"
        - type: glob
          property: RoleName
          value: "*SSMSession*"
        - type: exact
          property: RoleName
          value: "<security_scanner_role>"

accounts:
  "<env_account_id>":
    presets:
      - landing-zone
      - cool-baseline
```

**IMPORTANT:** The sample above is illustrative, NOT runnable as-is — it
does not enumerate every Control Tower-managed resource (VPC endpoints and
other Account Factory networking details vary by deployment), and resource
names and aws-nuke filter properties change over time. Obtaining the
canonical config from the DevSecOps team is a hard prerequisite of this
runbook; if it is unavailable, STOP and open a ticket with the DevSecOps
team rather than proceeding with the sample. The authoritative way to tune
the canonical config is the dry run in the next section, performed on a
freshly destroyed environment, with the DevSecOps team reviewing anything
unexpected in the kill list.

## 3. Dry run and review

1. Edit the config so the `accounts` key names the account you are
   sanitizing (the account ID of `env<X>`).
1. Run aws-nuke in its default dry-run mode using the sanitization profile:

    ```console
    aws-nuke run --config cool-aws-nuke.yaml --profile <sanitize_profile>
    ```

    **NOTE:** The `ProvisionAccount` role is scoped to provisioning and
    does not carry enumerate/delete permissions for every resource family
    this sweep targets; running under it produces access-denied noise and
    an unreliable verification. Use the dedicated sanitization role/profile
    designated by the DevSecOps team, which must have administrative access
    to the account.

    **NOTE:** aws-nuke requires the target account to have an IAM account
    alias and will prompt you to re-type it as a safety check.
1. Review the output. Every line is marked either `would remove` or
   `filtered`:
    1. Confirm that everything marked `filtered` is baseline (Control
       Tower, IAM Identity Center, COOL roles).
    1. Confirm that everything marked `would remove` is engagement debris.
       If you see anything you cannot identify, STOP and contact the
       DevSecOps team before proceeding.
1. If baseline resources appear in the `would remove` list, the config is
   out of date. Do NOT proceed; work with the DevSecOps team to update the
   canonical config first.

## 4. Sanitize the account

1. Re-run with deletion enabled:

    ```console
    aws-nuke run --config cool-aws-nuke.yaml --profile <sanitize_profile> --no-dry-run
    ```

1. Type the account alias when prompted to confirm.
1. Expect several minutes and some transient errors as aws-nuke works
   through resource dependencies. A single run is NOT guaranteed to remove
   everything: a resource can fail to delete because a dependent resource
   is removed later in the same run.
1. When it completes, review the run output for entries marked `failed`,
   access-denied, or otherwise non-removable. Every such entry must be
   either resolved (fix the cause and re-run) or explicitly reviewed with
   the DevSecOps team, approved for retention, and documented in the
   ticket — do NOT treat failures as ignorable noise.
1. Re-run the dry run from the previous section. The `would remove` list
   should now be empty — this, together with a failure-free deletion run,
   is the verification that the account is clean (within aws-nuke's
   resource-type coverage; see the note at the top of this page).
    1. If the list is not empty, re-run the deletion command; resources
       often become deletable once their dependents are gone. Repeat the
       delete/dry-run cycle until the dry run is empty. If it stops making
       progress, contact the DevSecOps team.
1. Check the DevSecOps team's list of aws-nuke-unsupported services and
   manually verify (and clean, if needed) anything the engagement used
   from that list.
1. Attach the final, empty dry-run output — plus documentation of any
   approved retained failures and manual checks — to the
   destroy-environment ticket.
1. Only after a clean verification run should the account be marked
   "AVAILABLE FOR USE" in the COOL environment tracking spreadsheet.

## Troubleshooting: CloudFormation stacks stuck in DELETE_FAILED

Stacks created with a service role (CDK apps are the common case — their
stacks run as a `cdk-<qualifier>-cfn-exec-role-*` role) can become
undeletable through normal means if that role was deleted or de-permissioned
before the stack — for example, when someone removes the CDK bootstrap
before the app stacks that depend on it. The symptoms in the stack events
are "security token included in the request is invalid" (role deleted) or
"not authorized to perform" errors (role stripped).

The fix is to re-drive the deletion with a working role instead of the
stack's broken one:

1. Retry the deletion, supplying an administrative CloudFormation service
   role to override the one recorded on the stack:

    ```console
    AWS_DEFAULT_REGION=us-east-1 AWS_PROFILE=cool-env<X>-provisionaccount aws cloudformation delete-stack --stack-name <stack_name> --role-arn <admin_capable_cfn_role_arn>
    ```

1. If the underlying resources were already deleted out-of-band, you may
   instead retain them (valid only from the `DELETE_FAILED` state; the
   logical IDs are listed in the stack's status reason):

    ```console
    AWS_DEFAULT_REGION=us-east-1 AWS_PROFILE=cool-env<X>-provisionaccount aws cloudformation delete-stack --stack-name <stack_name> --retain-resources <logical_id_1> <logical_id_2>
    ```

1. On newer AWS CLI versions, `--deletion-mode FORCE_DELETE_STACK` is a
   further fallback from the `DELETE_FAILED` state.
1. Delete app stacks before their CDK bootstrap/toolkit stack, never the
   reverse — deleting the bootstrap first is what creates this situation.
