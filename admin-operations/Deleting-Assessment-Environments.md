# Deleting Assessment Environments

**Purpose:** To destroy assessment environments in the COOL at the end of an
assessment.

**Prerequisites:** You MUST complete
[Admin Workstation Setup](Admin-Workstation-Setup.md) before proceeding.

You will receive a ticket relating to deleting the assessment environment.
Assessment data should already have been archived (see
[Archiving Assessment Data to the Analytic Enclave](Archiving-Assessment-Data-to-the-Analytic-Enclave.md))
before the environment is destroyed.

## Special cases (review BEFORE beginning the deletion process)

**Environments containing a terraformer instance:**

1. Contact the Fed Lead for the environment and confirm **in writing** (add
   it to the comments section of the ticket) that all infrastructure created
   with that terraformer has been destroyed by them.
1. Infrastructure destruction can be verified through the AWS Management
   Console by logging into the specific account and verifying all EC2, VPC,
   and other infrastructure have been deleted.

    **NOTE:** It is important to work closely with the Fed Lead to understand
    the types of infrastructure and **all regions** they deployed resources
    to, in order to verify that everything was deleted.
1. Only after this verification should you begin to delete an environment
   containing a terraformer instance.

## 1. Delete the PTR record(s)

1. Obtain info about the Elastic IP (EIP) using the AWS console:
    1. Log into the AWS console (Production or Staging-a).
    1. Switch your role to the appropriate sub-account (the account number of
       the environment whose PTR records you are going to delete), using
       `ProvisionAccount` as the IAM role name.
    1. Navigate to the EC2 Dashboard, then click the "Elastic IPs" sub-menu
       under "Network and Security" in the left margin.
    1. Locate the instance whose PTR record you want to remove and check the
       box to the left of its name.
    1. A summary menu opens at the bottom of the screen; one of the
       attributes shown is the EIP allocation ID (format `eipalloc-...`). You
       should also see the domain name associated with the PTR record you are
       trying to delete (in the "Reverse DNS Record" column).
1. Confirm the PTR record exists and matches the correct public IP:

    ```console
    AWS_DEFAULT_REGION=us-east-1 AWS_PROFILE=cool-env<X>-provisionaccount aws ec2 describe-addresses-attribute --allocation-ids eipalloc-<eipalloc_of_instance> --attribute domain-name
    ```

1. Reset the PTR record associated with the EIP:

    ```console
    AWS_DEFAULT_REGION=us-east-1 AWS_PROFILE=cool-env<X>-provisionaccount aws ec2 reset-address-attribute --allocation-id eipalloc-<eipalloc_of_instance> --attribute domain-name
    ```

    The PTR record is deleted and the status is set to `PENDING`.
1. Wait a minute or more, then check the status of the deleted PTR record by
   refreshing the AWS console. Once the reverse DNS record is no longer
   shown, the PTR record has been successfully deleted.

## 2. Revoke access in FreeIPA

1. Delete the FreeIPA user group associated with the assessment environment
   by following the
   [FreeIPA Group Management](FreeIPA-Group-Management.md#c-removing-user-groups)
   section entitled "Removing user groups."

## 3. Delete the Guacamole host in FreeIPA

1. Continue using your FreeIPA browser window from the previous step
   (Production: `https://ipa0.cool.example.gov/`, Staging-a:
   `https://ipa0.staging-a.cool.example.gov/`).
1. Under the "Identity" tab, click "Hosts".
1. Check the checkbox next to the host name for the Guacamole instance from
   the assessment environment being deleted (e.g.
   `guac.env0.cool.example.gov`).
1. Click the "Delete" button to bring up the "Remove hosts" dialog. DO NOT
   check the box in the dialog.
1. Click "Delete" to delete the Guacamole host.

**IMPORTANT:** Make sure that you are deleting the Guacamole **host** and NOT
the HBAC rule. HBAC rules should NEVER be deleted from FreeIPA.

## 4. Delete the environment

**NOTE:** Wherever `<environment_name>` is listed below, replace it with the
name of the environment you are deleting (e.g. `env0-production`).

1. In the COOL environment tracking spreadsheet, replace the status in the
   Current Uses column with "* DESTRUCTION IN PROGRESS *" and highlight it in
   red.
1. Pull in the latest repository changes:

    ```console
    cd ~/code/cisagov/cool-assessment-tfvars
    git pull
    cd ~/code/cisagov/cool-assessment-terraform
    git pull
    ```

**For COOL Production:**

1. If you were previously working on the Staging-a backend, switch to the
   production backend:

    ```console
    AWS_SHARED_CREDENTIALS_FILE=~/.aws/credentials terraform init -upgrade -reconfigure -backend-config=production.tfconfig
    ```

1. If you did not need to reconfigure the backend, simply initialize
   Terraform: `terraform init -upgrade`
1. Display the available Terraform workspaces: `terraform workspace list`
1. Select the workspace of the environment you are planning to destroy:
   `terraform workspace select env<X>-production`
1. Run the apply script:

    ```console
    AWS_SHARED_CREDENTIALS_FILE=~/.aws/credentials AWS_DEFAULT_REGION=us-east-1 ./terraform_apply.sh -var-file=<environment_name>-production.tfvars
    ```

1. When prompted to continue, examine the Terraform output for correctness.
1. If the output is correct, type `yes` to apply the changes (type `no` to
   exit if you are unsure of the output).

    **NOTE:** The apply script is segmented into three different parts, so
    you will examine and approve three Terraform plans.
1. After the apply script has completed without errors, run the destroy
   script:

    ```console
    AWS_SHARED_CREDENTIALS_FILE=~/.aws/credentials AWS_DEFAULT_REGION=us-east-1 ./terraform_destroy_all.sh -auto-approve -var-file=<environment_name>-production.tfvars
    ```

1. If Terraform reported any errors other than the EIP release error, repeat
   the previous step. If the errors persist, contact a member of the
   DevSecOps team for assistance.
1. If you receive no errors, the destroy script completed successfully and
   you may proceed.
1. Run `terraform state list` to verify that no objects or infrastructure are
   left in the environment. The command should return no output. If it
   returns output, notify a member of the DevSecOps team so the objects can
   be manually removed.

**For COOL Staging-a:**

1. Initialize the Staging-a backend:

    ```console
    AWS_SHARED_CREDENTIALS_FILE=~/.aws/cool-staging-a-credentials terraform init -upgrade -reconfigure -backend-config=staging-a.tfconfig
    ```

1. If you did not need to reconfigure the backend, simply initialize
   Terraform: `terraform init -upgrade`
1. Display the available Terraform workspaces:

    ```console
    AWS_SHARED_CREDENTIALS_FILE=~/.aws/cool-staging-a-credentials terraform workspace list
    ```

1. Select the workspace of the environment you are planning to destroy:

    ```console
    AWS_SHARED_CREDENTIALS_FILE=~/.aws/cool-staging-a-credentials terraform workspace select env<X>-staging-a
    ```

1. Run the apply script:

    ```console
    AWS_SHARED_CREDENTIALS_FILE=~/.aws/cool-staging-a-credentials AWS_DEFAULT_REGION=us-east-1 ./terraform_apply.sh -var-file=<environment_name>-staging-a.tfvars -var-file=windows-ami-production.tfvars
    ```

1. When prompted to continue, examine the Terraform output for correctness.
1. If the output is correct, type `yes` to apply the changes (type `no` to
   exit if you are unsure of the output).

    **NOTE:** The apply script is segmented into three different parts, so
    you will examine and approve three Terraform plans.
1. After the apply script has completed without errors, run the destroy
   script:

    ```console
    AWS_SHARED_CREDENTIALS_FILE=~/.aws/cool-staging-a-credentials AWS_DEFAULT_REGION=us-east-1 ./terraform_destroy_all.sh -auto-approve -var-file=<environment_name>-staging-a.tfvars
    ```

1. If Terraform reported any errors other than the EIP release error, repeat
   the previous step. If the errors persist, contact a member of the
   DevSecOps team for assistance.
1. If you receive no errors, the destroy script completed successfully and
   you may proceed.
1. Run `terraform state list` to verify that no objects or infrastructure are
   left in the environment. The command should return no output. If it
   returns output, notify a member of the DevSecOps team so the objects can
   be manually removed.

## 5. Update the Terraform variables file

1. In the COOL environment tracking spreadsheet, determine which environment
   you are deleting.
1. Navigate to the `cool-assessment-tfvars` repository on GitHub and choose
   the file associated with that environment.
1. Click Edit and remove the assessment-specific variables (domain, instance
   counts, Nessus key, etc.).

    **NOTE:** It is easiest to copy the text of an empty tfvars file (one
    that has already been sanitized).
1. Double-check the file once complete to ensure there are no extra
   characters or spaces, as this may affect the terraforming process.
1. In the Commit Changes area at the bottom of the file, enter:
   `Sanitize file at end of assessment.`
1. Click Commit Changes.

## 6. Closeout tasks

1. In the COOL environment tracking spreadsheet, replace the status in the
   Current Uses column with "AVAILABLE FOR USE" and make the cell color white
   or clear.
1. In the destroy-environment sub-ticket, enter the following text in the
   comment section: `Environment destruction complete.`
1. Once you close the sub-ticket, navigate to the parent ticket and close it
   out as well.
