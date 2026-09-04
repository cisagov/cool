# Support and Troubleshooting Guide

**NOTE:** This guide outlines the basic types of support and troubleshooting
requests a COOL admin may receive through the ticketing system, and lays out
generic guidance on how to approach solving them. This is NOT a step-by-step
guide.

## VPN and environment access issues

**Is the user's PIV inserted and being read correctly?**

- Read the certificate: `pkcs15-tool --read-certificate 01`
- Check that the PKCS#11 object is still present — on macOS at
  `/Library/OpenSC/lib/opensc-pkcs11.so`; on Linux, the OpenSC module is
  typically at `/usr/lib/x86_64-linux-gnu/opensc-pkcs11.so` or
  `/usr/lib64/opensc-pkcs11.so`
- In the VPN client, check the connection's certificate retrieval setting
  (Edit Connection → Auth → Retrieval → "Use Cert from below" → Detect)

**Is the SSO script completing correctly, and is the user being issued a
Kerberos ticket?**

- Run `klist` in the command prompt.
- If the command does not return a credential (for Production or Staging-a),
  they do not have a Kerberos ticket and have not connected to the VPN
  correctly.
- Have the user restart their workstation a few times.
- Have the user fully close and restart the VPN client.

**SSO issues — manually log out of the Kerberos realm, reset the cache, and
log back in:**

For Production:

```console
app-sso -d COOL.EXAMPLE.GOV
app-sso -r COOL.EXAMPLE.GOV
app-sso -a COOL.EXAMPLE.GOV -u <firstname.lastname>
```

For Staging-a:

```console
app-sso -d STAGING-A.COOL.EXAMPLE.GOV
app-sso -r STAGING-A.COOL.EXAMPLE.GOV
app-sso -a STAGING-A.COOL.EXAMPLE.GOV -u <firstname.lastname>
```

**NOTE:** `app-sso` is the macOS single sign-on utility. On Linux clients,
destroy and re-obtain the ticket directly: `kdestroy`, then
`kinit <firstname.lastname>@COOL.EXAMPLE.GOV` (or the Staging-a realm).

**User can connect to the VPN but cannot access specific environments:**

- Check the user's account in FreeIPA and ensure they have been provided
  access to the environment(s).
- Ensure that you (as the COOL admin) can access the environment.
- Have the user try a different web browser (e.g. Firefox or Safari).
- Have the user clear their browser cache.

## Users unable to connect to a Guacamole environment

This can be caused by a variety of local and COOL-related issues. Try to
isolate the issue with the steps below:

1. As an admin, first try to access the environment yourself using multiple
   browsers.
1. If a specific user cannot access the environment but other users (or
   admins) can: have the user try a different browser (e.g. Safari or
   Firefox) and clear their browser cache.
1. If all users (including you) cannot access the environment, try stopping
   and restarting the Apache service. SSM into the specific Guacamole
   instance, then:

    ```console
    sudo systemctl stop apache2
    sudo systemctl restart apache2
    ```

    Check whether the environment is accessible again.

**The Guacamole certificate could be the reason users are unable to
connect** (expired, corrupted, missing, or an archive-directory conflict):

- If you get an error like `archive directory exists for <guac-name>` while
  renewing the cert, delete the archive directory from the certificates
  bucket, then recreate and redeploy the cert.
- Sometimes the cert shows active and up to date, but if the cert is not in
  the S3 bucket the Guacamole instance won't connect. Recreate and redeploy
  the cert:

    ```console
    cd ~/code/cisagov/certboto-docker
    docker-compose -f my-cool-docker-compose.yml run certboto certonly -d guac.env<X>.cool.example.gov
    ```

- Find the Guacamole instance ID:

    ```console
    cd ~/code/cisagov/cool-assessment-terraform
    terraform workspace select env<X>-production
    terraform state show aws_instance.guacamole | grep id | head -n1
    ```

- SSM into the Guacamole instance.

    For Production:

    ```console
    AWS_SHARED_CREDENTIALS_FILE=~/.aws/credentials AWS_PROFILE=cool-env<X>-startstopssmsession AWS_DEFAULT_REGION=us-east-1 aws ssm start-session --target i-<guac_instance_id> --document-name SSMSessionManagerRunShell
    ```

    For Staging-a:

    ```console
    AWS_SHARED_CREDENTIALS_FILE=~/.aws/cool-staging-a-credentials AWS_PROFILE=cool-env<X>-startstopssmsession AWS_DEFAULT_REGION=us-east-1 aws ssm start-session --target i-<guac_instance_id> --document-name SSMSessionManagerRunShell
    ```

- Install the certificate, restart Apache, and check the cert dates:

    ```console
    sudo /var/lib/cloud/instance/scripts/install-certificates.py; sudo apache2ctl restart; sudo openssl x509 -dates -in /var/guacamole/httpd/ssl/self.cert
    ```

## Stopping/restarting individual instances

**NOTE:** Fed Leads have the ability to stop and restart instances in their
environments themselves. Only follow the steps below if they are NOT able to
restart the instance:

1. Log into the AWS Management Console with your COOL AWS credentials.
1. Navigate to the proper sub-account (the list of COOL sub-accounts is in
   the COOL environment tracking spreadsheet).
1. Consult the AWS documentation on how to stop and start an instance and
   follow those instructions.
1. Once you have stopped and restarted the instance and verified this in the
   AWS Management Console, respond to the ticket submitter in the comments
   section indicating that action has been taken.

**NOTE:** AWS maintains an enormous knowledge base for the AWS Management
Console. Consult it as part of your normal duties to maintain proficiency.

## Terraform tainting of Elastic IPs (EIPs)

Used when a ticket requests a **new** Elastic IP for an assessment
environment (e.g. an IP has been burned/blocklisted):

1. Receive a ticket requesting a new EIP for an assessment environment.
1. Delete the PTR record associated with the EIP of the instance by following
   Step 1 of
   [Deleting Assessment Environments](Deleting-Assessment-Environments.md).
1. Taint the existing EIP:
    - Obtain the resource name: `terraform state list | grep eip`
      (it will most likely be a GoPhish instance, e.g.
      `aws_eip.gophish[0]`).
    - Taint the EIP: `terraform taint aws_eip.<resource_name>`
1. Run the apply script:

    ```console
    AWS_SHARED_CREDENTIALS_FILE=~/.aws/credentials AWS_DEFAULT_REGION=us-east-1 ./terraform_apply.sh -var-file=<environment_name>-production.tfvars
    ```

1. Review each step of the plan carefully prior to accepting, to ensure that
   the only changes Terraform wants to make are associated with the EIP.
1. Type `yes` if everything looks correct after reviewing each step.
1. Obtain the new EIP of the instance, provide it to the Fed Lead, and
   request an A record be set by following the "Provide IPs for Instances"
   step of
   [Provisioning Assessment Environments](Provisioning-Assessment-Environments.md).
1. Once the A record is set, create the PTR record by following the "PTR
   Record Creation" step of the same runbook.
1. Respond to the ticket submitter in the comments section indicating that
   action has been taken, then close out the ticket.

## EFS provisioned but not mounted to Kali instances (manual mounting)

Replace all environment numbers and ID numbers with the appropriate
information.

1. Determine the host AWS instance ID:

    ```console
    terraform workspace select env<X>-production
    terraform state show aws_instance.kali[0] | grep id | head -n1
    ```

1. Use SSM to open a shell on the instance:

    ```console
    AWS_PROFILE=cool-env<X>-startstopssmsession AWS_DEFAULT_REGION=us-east-1 aws ssm start-session --target i-<instance_id> --document-name SSMSessionManagerRunShell
    ```

1. (Optional, but recommended) Start up a bash shell: `bash`
1. Run `findmnt` and look for an entry similar to:

    ```console
    /share  127.0.0.1:/  nfs4  rw,relatime,vers=4.1,rsize=1048576,wsize=1048576,namlen=255,hard
    ```

    **NOTE:** If you do not see a similar entry, the EFS did not mount to the
    instance.
1. Mount the EFS: `sudo mount -a`

    **NOTE:** If the command is successful there will be no output.
1. Re-run `findmnt` and confirm the `/share` entry now appears. If it does,
   you are done — exit the SSM shell.

## Windows Commando instance not mounted to share

1. Try stopping and then restarting the Windows Commando instance through the
   AWS Management Console.
1. Ask the Fed Lead to verify that this solved the issue.
1. If stopping/restarting the instance did not work, contact a member of the
   DevSecOps team for assistance, as there may be an issue with the samba
   instance.

## Firewall rules lockouts

If an operator has locked themselves out of an instance with firewall rules,
the simplest approach is to coordinate with the ticket reporter and ask if it
is OK to disable the firewall. After disabling it, they should be able to
reconnect via Guacamole and fix their firewall rules so that ports 22 (SSH)
and 5901 (VNC) are allowed from the Guacamole instance or subnet. Allowing
22 and 5901 in from anywhere is acceptable as long as those ports are not
publicly exposed to the internet.

- See if the firewall is running and list the rules: `sudo ufw status numbered`
- Stop the firewall: `sudo ufw disable`
- Start the firewall: `sudo ufw enable`

## Hostname changed from default; can no longer connect through Guacamole

1. Determine the host AWS instance ID:

    ```console
    terraform workspace select env<X>-production
    terraform state show aws_instance.teamserver[0] | grep id | head -n1
    ```

1. Use SSM to open a shell on the instance:

    ```console
    AWS_PROFILE=cool-env<X>-startstopssmsession AWS_DEFAULT_REGION=us-east-1 aws ssm start-session --target i-<instance_id> --document-name SSMSessionManagerRunShell
    ```

1. (Optional, but recommended) Start up a bash shell: `bash`
1. Set the hostname back to the default, substituting the internal IP of the
   instance (dashes, not dots):

    ```console
    sudo hostnamectl set-hostname ip-<internal-ip-with-dashes>.ec2.internal
    ```

1. Restart the VNC service:

    ```console
    sudo systemctl restart vncserver@1.service
    ```

1. See if you can now interact with the instance through Guacamole. If the
   issue is fixed, exit both bash and the SSM session.

## Nessus instance asking for activation code / registration unsuccessful

If an environment's Nessus instance is asking for an activation code:

1. Make sure the license has been reset (contact the DevSecOps team POC for
   Nessus licenses).
1. Sign into the instance via SSM and run:

    ```console
    /opt/nessus/sbin/nessuscli fetch --register-only <activation_code>
    ```

## COOL CloudWatch alarms

If you notice a CloudWatch alarm similar to:
`Status Check Alarm: "ec2_instance_status_check_i-<instance_id>" in US East (N. Virginia)`:

1. Locate the account number and instance ID within the alarm message.
1. Open the AWS console and switch roles to the account the alarm originated
   from.
1. Examine the affected instance to determine if it is in a degraded/alarm
   state (you can also double-check whether it is accessible within its
   Guacamole environment).
1. If the instance is unreachable, stop the instance.
1. Restart the instance once it is fully stopped.
1. Recheck the instance to ensure that it is now reachable (through
   Guacamole).

## Corrupted certificate data affecting certboto-docker

An unknown issue can occur where the data for a particular certificate in the
S3 bucket gets corrupted. When that happens, normal certboto-docker commands
fail due to the warning emitted by `rebuild-symlinks.py`.

To reproduce the issue:

```console
docker compose run certboto certificates
```

Output will resemble:

```console
Syncing certbot configs from <certificates_bucket>
Rebuilding symlinks in /etc/letsencrypt
WARNING Could not find a matching entry in the archive!
```

To fix the issue:

1. Start up a container:

    ```console
    docker compose -f my-cool-docker-compose.yml run --entrypoint /bin/sh certboto
    ```

1. Sync data from the certificates bucket:

    ```console
    AWS_PROFILE=cool-dns-certificatesbucketfullaccess aws s3 sync "s3://<certificates_bucket>" /etc/letsencrypt
    ```

1. Determine the corrupted certificate:

    ```console
    ./rebuild-symlinks.py --log-level=info /etc/letsencrypt 2>&1 | grep -B2 WARNING
    ```

1. Delete the corrupted certificate:

    ```console
    certbot delete --cert-name=<corrupted_cert_name>
    ```

1. Sync the cleaned-up list back to the bucket:

    ```console
    AWS_PROFILE=cool-dns-certificatesbucketfullaccess aws s3 sync --delete /etc/letsencrypt "s3://<certificates_bucket>"
    ```

After that, certboto-docker commands should work normally and a fresh
certificate can be generated.

## Quick commands

**Check cloud-init logs on an instance:**

```console
sudo cat /var/log/cloud-init-output.log
```

**Restart the VNC service on an instance:**

```console
sudo systemctl restart vncserver@1.service
```
