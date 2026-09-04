# Provisioning Assessment Environments

**Purpose:** To provision new assessment environments in the COOL.

**Prerequisites:** You MUST complete
[Admin Workstation Setup](Admin-Workstation-Setup.md) before proceeding.

**Tickets:** You will receive an "Assessment Lab Request" umbrella ticket. At
the bottom of the umbrella ticket you will see a number of sub-tickets. The
ones relevant to provisioning are:

- "Nessus Instance License Request"
- "Assign Operator"
- "Environment Request - Provide IP Addresses / Hostnames in Description
  Below"

## 1. Create/renew SSL certificates for mail servers (if needed)

**NOTE:** This step is usually necessary for RPTs and RVAs only. The domain
names are provided in the body of the "Assessment Lab Request" ticket (there
is usually a primary and a secondary domain). If you do not see the domains,
ask the Fed Lead to provide them.

1. If you have been given permissions to create COOL SSL certificates,
   complete
   [Creating SSL Certificates for Assessment Mail Servers](Creating-SSL-Certificates-for-Assessment-Mail-Servers.md).
1. If you do not have those permissions, contact a member of the DevSecOps
   team who can perform this step.
1. Once this task is complete, the DevSecOps team POC will add a comment in the
   ticket stating that the domain certificates were created/renewed.

## 2. Assign a Nessus license (if needed)

**NOTE:** This step is usually necessary for RPTs and RVAs (or any
environment requiring a Nessus instance).

1. If you have been given permissions to assign Nessus licenses, assign a
   Nessus license from the appropriate assessment group (usually RPTs or
   RVAs) to the assessment.
1. If you do not have those permissions, contact a member of the DevSecOps
   team who can perform this step.
1. Close the "Nessus Instance License Request" sub-ticket.

## 3. Create the IPA group and assign operators

**NOTE:** Operator names are generally provided in the body of the umbrella
ticket. If names are not provided, they may not be known to the Fed Lead yet;
a ticket to add specific operators may be submitted later. When in doubt, ask
the Fed Lead.

1. Open the COOL environment tracking spreadsheet and choose an open
   environment (one with the "Current Uses" column marked "AVAILABLE FOR
   USE"). Fill in the details of the assessment (assessment type, number, and
   name of Fed Lead) and label the status "COMING SOON *".
1. Address the "Assign Operator" sub-ticket by following
   [FreeIPA Group Management](FreeIPA-Group-Management.md). This sets up the
   FreeIPA user group and identifies the specific assessment environment in
   which to provision the assessment resources.

## 4. Update the Terraform variables file

1. Using the COOL environment tracking spreadsheet, determine which
   environment you will be terraforming (chosen during the previous step).
1. Navigate to the `cool-assessment-tfvars` repository on GitHub and choose
   the file associated with that environment.
1. Click Edit and input the variables provided in the tickets (assessment
   number, type, domains, instance counts, Nessus key, etc.).
1. While editing this `.tfvars` file, you must also specify the inbound ports
   for the associated instances (the current baseline per instance type is
   documented in the private `cool-assessment-tfvars` repository). If custom
   ports are required, they should be
   included in the ticket (when in doubt, ask the Fed Lead). If custom ports
   are not specified (normally they are not), you can copy the inbound ports
   from the `.tfvars` file of an active assessment environment of the same
   type. For example, if you are terraforming an RPT environment, look at the
   `.tfvars` file of an active RPT.
1. Double-check the file once complete to ensure there are no extra
   characters or spaces, as this may affect the terraforming process.
1. In the Commit Changes area at the bottom of the file, enter a useful
   message describing what you changed (e.g. "Add domain, Nessus key,
   instance counts, inbound ports") and click Commit Changes.

## 5. Provision the new environment

**NOTE:** Wherever `<environment_name>` is listed below, replace it with the
name of the environment you are provisioning (e.g. `env0-production`).

Pull in the latest repository changes:

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
1. Select the workspace of the environment you are provisioning:
   `terraform workspace select env<X>-production`
1. Run the apply script:

    ```console
    AWS_SHARED_CREDENTIALS_FILE=~/.aws/credentials AWS_DEFAULT_REGION=us-east-1 ./terraform_apply.sh -var-file=<environment_name>-production.tfvars
    ```

1. When prompted to continue, examine the Terraform output for correctness.
1. If the output is correct, type `yes` to apply the changes (type `no` to
   exit if you are unsure of the output).

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

1. Select the workspace of the environment you are provisioning:

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

**NOTE:** The apply script is segmented into different parts, so you will
examine and approve multiple Terraform plans before the script completes.

## 6. Perform FreeIPA and Guacamole actions

Each of these post-apply actions consists of several steps of its own; the
detailed steps are in the following linked sub-tasks:

1. [Joining a New Guacamole Host to the FreeIPA Domain](Joining-a-New-Guacamole-Host-to-the-FreeIPA-Domain.md)
1. [Creating or Updating a FreeIPA HBAC Rule](Creating-or-Updating-a-FreeIPA-HBAC-Rule.md)
1. [Verifying the Nessus Setup Script](Verifying-the-Nessus-Setup-Script.md)

## 7. Provide IPs for instances — request A records

1. Obtain the public IP addresses of the instances:
    1. Log into the AWS console (Production or Staging-a).
    1. Switch your role to the appropriate sub-account (the account number of
       the environment you provisioned), using `ProvisionAccount` as the IAM
       role name.
    1. Navigate to the EC2 Dashboard.
    1. Locate and click on an instance in the "Instances" menu.
    1. Check the box to the left of the instance name.
    1. Locate and copy the "Public IP Address" of the instance (shown in the
       bottom menu once you check the box).
1. Obtain the public IPs of the key instances to provide to the Fed Lead
   (normally 5 instances total):
    - Teamservers (normally 2 total)
    - Nessus (normally 1 total)
    - GoPhish (normally 2 total)
1. In the "Environment Request - Provide IP Addresses / Hostnames" sub-ticket
   description, paste the appropriate statement below, filled in with the
   environment number, public IPs, and domains.

    **For RPTs:**

    > Your new RPT environment is ready at:
    > https://guac.env\<environment number\>.cool.example.gov
    > The public IP for teamserver0.env\<environment number\> is: \<teamserver0 IP address\>
    > The public IP for teamserver1.env\<environment number\> is: \<teamserver1 IP address\>
    > The public IP for nessus0.env\<environment number\> is: \<nessus IP address\>
    > The public IP for gophish0.env\<environment number\> is: \<gophish0 IP address\>
    > The public IP for gophish1.env\<environment number\> is: \<gophish1 IP address\>
    > Please create the A record for: \<primary email domain from ticket\> -> \<teamserver0 IP address\> and \<secondary email domain from ticket\> -> \<teamserver1 IP address\> and let us know when that is done so that we can create the PTR record.

    **For RVAs:**

    > Your new RVA environment is ready at:
    > https://guac.env\<environment number\>.cool.example.gov
    > The public IP for teamserver0.env\<environment number\> is: \<teamserver0 IP address\>
    > The public IP for teamserver1.env\<environment number\> is: \<teamserver1 IP address\>
    > The public IP for nessus0.env\<environment number\> is: \<nessus IP address\>
    > The public IP for gophish0.env\<environment number\> is: \<gophish0 IP address\>
    > The public IP for gophish1.env\<environment number\> is: \<gophish1 IP address\>
    > Please create the A record for: mail.\<primary email domain from ticket\> -> \<gophish0 IP address\> and mail.\<secondary email domain from ticket\> -> \<gophish1 IP address\> and let us know when that is done so that we can create the PTR record.

1. Mark the "Environment Request - Provide IP Addresses / Hostnames"
   sub-ticket as closed.

    **NOTE:** This sends a message to the Fed Lead to update the DNS records
    for the domains assigned to the assessment. This may take time (hours or
    days).
1. In the COOL environment tracking spreadsheet, replace the status in the
   Current Uses column with "* AWAITING A RECORD *".
1. Once the A records are created, a "Create PTR Record" sub-ticket will
   appear as part of the umbrella ticket. This is your signal to proceed to
   create the PTR records.

## 8. PTR record creation

You will receive a "PTR Record Creation" sub-ticket. Complete the steps below
to set up a PTR record via the CLI.

**NOTE:** You will need the EIPs for either both teamserver instances or both
GoPhish instances, depending on whether the assessment is an RPT or an RVA.
You are setting the PTR (reverse DNS) records for the instances that you
requested the A records for in the previous step.

1. Obtain info about the Elastic IP (EIP) using the AWS console:
    1. Log into the AWS console (Production or Staging-a).
    1. Switch your role to the appropriate sub-account (the account number of
       the environment), using `ProvisionAccount` as the IAM role name.
    1. Navigate to the EC2 Dashboard, then click the "Elastic IPs" sub-menu
       under "Network and Security" in the left margin.
    1. Locate the instance and check the box to the left of its name.
    1. A summary menu opens at the bottom of the screen; one of the
       attributes shown is the EIP allocation ID (in the format
       `eipalloc-...`).
1. Confirm the A record exists and matches the correct public IP. In a
   terminal, perform an nslookup for each domain (primary and secondary):

    ```console
    nslookup <primary_domain_name>
    nslookup <secondary_domain_name>
    ```

1. Check the current PTR record for the EIP:

    ```console
    AWS_DEFAULT_REGION=us-east-1 AWS_PROFILE=cool-env<X>-provisionaccount aws ec2 describe-addresses-attribute --allocation-ids eipalloc-<eipalloc_of_instance> --attribute domain-name
    ```

    **NOTE:** The output should be `"Addresses": []` since the PTR record has
    not yet been set.
1. Set the PTR record for the EIP:

    ```console
    AWS_DEFAULT_REGION=us-east-1 AWS_PROFILE=cool-env<X>-provisionaccount aws ec2 modify-address-attribute --allocation-id eipalloc-<eipalloc_of_instance> --domain-name <domain_name>
    ```

    The output of the command should show `"Status": "PENDING"`.

    **NOTE:** You will need to wait a minute or two for the PTR to be set.
1. Confirm the PTR record was set (after waiting a few minutes):

    ```console
    AWS_DEFAULT_REGION=us-east-1 AWS_PROFILE=cool-env<X>-provisionaccount aws ec2 describe-addresses-attribute --allocation-ids eipalloc-<eipalloc_of_instance> --attribute domain-name
    ```

    **NOTE:** The output should show the public IP, the allocation ID, and
    the PTR record (set to the domain you provided in the previous step).
1. Confirm the PTR record was set via DNS:

    ```console
    dig +noall +ans -x <publicIP_of_instance>
    ```

    **NOTE:** The output should be the IP and the PTR record.
1. Close out the "PTR Record Creation" sub-ticket: add the text "PTR records
   created" to the sub-ticket comments section and close it out.

## 9. Closeout tasks

1. In the COOL environment tracking spreadsheet, delete the status text in
   the Current Uses column, leaving only the assessment number and the name
   of the requestor (e.g. `RPT - RV1234 (Jane Doe)`), and set the cell
   background to clear.
