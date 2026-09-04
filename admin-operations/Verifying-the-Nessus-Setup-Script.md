# Verifying the Nessus Setup Script

These steps are performed as part of the post-apply process when building an
assessment environment (see
[Provisioning Assessment Environments](Provisioning-Assessment-Environments.md)).

You will need to replace all environment numbers and ID numbers with the
appropriate information.

## Steps

1. Obtain the Nessus instance ID:
    1. Log into the AWS console (Production or Staging-a).
    1. Switch your role to the appropriate sub-account (the account number of
       the environment you provisioned), using `ProvisionAccount` as the IAM
       role name.
    1. Navigate to the EC2 Dashboard.
    1. Locate and click on the Nessus instance in the "Instances" menu.
    1. Check the box to the left of the instance name.
    1. Locate and copy the "Instance ID" of the Nessus instance (shown in the
       bottom menu once you check the box).
1. Use SSM to open a shell on the Nessus instance:

    ```console
    AWS_PROFILE=cool-env<X>-startstopssmsession AWS_DEFAULT_REGION=us-east-1 aws ssm start-session --target <instance_id> --document-name SSMSessionManagerRunShell
    ```

1. Verify that the Nessus admin user was successfully created:

    ```console
    sudo grep "admin user" /var/log/cloud-init-output.log
    ```

    - If the output is `Admin user successfully created`, continue to the
      next step.
    - If the output is
      `Admin username is empty; skipping creation of admin user`, the Nessus
      setup script did not run as expected. Re-run it:

        ```console
        sudo /var/lib/cloud/instance/scripts/nessus-setup.sh
        ```

        If everything was successful, you should see
        `Admin user successfully created` at the end of the output. If you
        still do not see that message, contact a member of the DevSecOps team
        for further troubleshooting assistance.
1. Restart the nessusd service and confirm it is running:

    ```console
    sudo systemctl restart nessusd
    sudo systemctl status nessusd
    ```

1. Verify that Nessus was successfully registered:

    ```console
    sudo grep "successfully registered" /var/log/cloud-init-output.log
    ```

    - If the output is `Nessus successfully registered`, continue to the next
      step.
    - If you do not see that output, it most likely means there is a problem
      with the activation code that was used. Confirm that the code is valid
      and that the correct code was put in the Terraform variables file when
      this environment was provisioned. When you update the activation code
      in the Terraform variables file, you can re-run the `terraform apply`
      and it will recreate the Nessus instance. After that completes, perform
      these validation steps again.
1. Log out of the Nessus instance:

    ```console
    exit
    ```
