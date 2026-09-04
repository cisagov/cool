# Joining a New Guacamole Host to the FreeIPA Domain

These steps are performed as part of the post-apply process when building an
assessment environment (see
[Provisioning Assessment Environments](Provisioning-Assessment-Environments.md)).

You will need to replace all environment numbers and ID numbers with the
appropriate information.

**Before starting**, verify that the Guacamole host is not already joined to
the FreeIPA domain by browsing to
`https://ipa0.cool.example.gov/ipa/ui/#/e/host/search` (or
`https://ipa0.staging-a.cool.example.gov/ipa/ui/#/e/host/search` for
Staging-a) and confirming that the host you want to join is not already in
the list of hosts.

## Steps

1. Obtain the Guacamole instance ID:
    1. Log into the AWS console (Production or Staging-a).
    1. Switch your role to the appropriate sub-account (the account number of
       the environment you provisioned), using `ProvisionAccount` as the IAM
       role name.
    1. Navigate to the EC2 Dashboard.
    1. Locate and click on the Guacamole instance in the "Instances" menu.
    1. Check the box to the left of the instance name.
    1. Locate and copy the "Instance ID" of the Guacamole instance (shown in
       the bottom menu once you check the box).
1. Use SSM to open a shell on the Guacamole instance:

    ```console
    AWS_PROFILE=cool-env<X>-startstopssmsession AWS_DEFAULT_REGION=us-east-1 aws ssm start-session --target <instance_id> --document-name SSMSessionManagerRunShell
    ```

1. Ensure that the Guacamole certificate is installed:

    ```console
    sudo /var/lib/cloud/instance/scripts/install-certificates.py
    ```

    **NOTE:** If the script runs normally, there will not be any output.
1. Use `sudo kinit` to get Kerberos credentials (replace `first.last` with
   your FreeIPA username, which must have administrative privileges):

    ```console
    sudo kinit first.last@COOL.EXAMPLE.GOV
    ```

    At the password prompt, enter your admin user's FreeIPA password.

    For Staging-a instances, use:

    ```console
    sudo kinit first.last@STAGING-A.COOL.EXAMPLE.GOV
    ```

1. Set up FreeIPA on the instance:

    ```console
    sudo /usr/local/sbin/00_setup_freeipa.sh
    ```

    Answer the prompts as follows:
    - `Continue to configure the system with these values? [no]:` enter `yes`
    - `User authorized to enroll computers:` enter your username
      (`first.last`), which must have administrative privileges.
    - `Password for first.last@COOL.EXAMPLE.GOV:` enter your admin user's
      FreeIPA password.
1. Set up the Guacamole HTTP service:

    ```console
    sudo /usr/local/sbin/01_setup_http_service.sh nom
    ```

1. Set up the Guacamole service:

    ```console
    sudo /usr/local/sbin/02_setup_guacamole_services.sh
    ```

1. Destroy your Kerberos ticket:

    ```console
    sudo kdestroy
    ```

1. Log out of the Guacamole instance:

    ```console
    exit
    ```

1. Verify that the host has been added by browsing to
   `https://ipa0.cool.example.gov/ipa/ui/#/e/host/search` (or
   `https://ipa0.staging-a.cool.example.gov/ipa/ui/#/e/host/search` for
   Staging-a) and confirming that the host is now present in the list of
   hosts (you may have to refresh the page).
