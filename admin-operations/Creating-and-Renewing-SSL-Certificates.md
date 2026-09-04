# Creating and Renewing SSL Certificates

**Purpose:** To create or renew SSL certificates for COOL service domains
using `certboto-docker`. This runbook is needed if you are assigned to manage
SSL certs for the COOL.

## A. Prerequisites

1. You have been issued an administrator workstation.
1. You are assigned to perform COOL SSL creation and renewal tasks. Your
   section lead notifies the DevSecOps team in writing that you are assigned to
   perform these duties.
1. You have been subscribed to the COOL certificates email distribution list
   in order to receive automated notifications when SSL certificates are
   close to expiration. Notify the DevSecOps team to have them subscribe you.
1. You have a recent version of Python installed (a virtual environment
   manager such as pyenv is recommended).
1. Install the latest AWS Command Line Interface (CLI) for your platform,
   following the AWS documentation.
1. You have `docker-compose` installed on your local system.

    **NOTE:** Installing the Docker Desktop app from the Docker website also
    installs docker-compose and the engine.

    **NOTE:** Always check that Docker is running on your workstation. If it
    is not running, start the Docker app or you will not be able to perform
    certificate renewals.
1. You have checked out the `cisagov/certboto-docker` repository into a
   directory called `certboto-docker`, following the instructions under the
   "Install" section of its README only.
1. You have completed the certboto-docker first-time setup:
    1. Open a terminal and navigate to your certboto-docker directory:
       `cd certboto-docker`
    1. Create a new Docker Compose file called `my-cool-docker-compose.yml`,
       replacing values within `<>` appropriately. For configuration details,
       refer to the certboto-docker README
       (`https://github.com/cisagov/certboto-docker#readme`).

## B. Renewing a certificate for a single domain

1. You receive certificate expiration notices in the form of emails from the
   "Let's Encrypt Expiry Bot" to the COOL certificates email distribution
   list. A notice may contain one or more domains depending on the timing of
   the certificate expiration. Notices are sent 20 days before expiration.

    **IMPORTANT:** You can only renew one certificate at a time with the
    current process, so if you receive a notification with two domains
    requiring renewal, you must renew them separately.
1. Open a terminal and navigate to your certboto-docker directory:
   `cd ~/code/cisagov/certboto-docker`
1. Double-check the certificate expiration dates:

    ```console
    docker-compose -f my-cool-docker-compose.yml run certboto certificates
    ```

1. Generate a new certificate:

    ```console
    docker-compose -f my-cool-docker-compose.yml run certboto certonly -d <fully_qualified_domain_name>
    ```

    If there are no errors, the new certificate data will be copied to the S3
    bucket specified in `my-cool-docker-compose.yml`.
1. Re-run the command from Step 3 and verify that the domain requiring
   renewal now shows an expiration of 89 days.
1. Forward the certificate expiration notice, along with a confirmation that
   the certificates have been renewed, to the COOL certificates email
   distribution list so that the DevSecOps team can deploy the new
   certificates.
1. Once the certificates have been deployed by the DevSecOps team, you should
   receive an email confirmation that the deployment was successful.

Congratulations! You have successfully renewed SSL certs for a single domain.

## C. Renewing a certificate for multiple domains

**IMPORTANT:** This section is only necessary for a small number of
multi-domain certificates covering internal service domains.

1. You receive certificate expiration notices as described in Section B.
1. Open a terminal and navigate to your certboto-docker directory:
   `cd ~/code/cisagov/certboto-docker`
1. Double-check the certificate expiration dates:

    ```console
    docker-compose -f my-cool-docker-compose.yml run certboto certificates
    ```

1. Generate a new certificate covering all of the domains:

    ```console
    docker-compose -f my-cool-docker-compose.yml run certboto certonly -d <fully_qualified_domain_name_1> -d <fully_qualified_domain_name_2> -d <fully_qualified_domain_name_n>
    ```

    If there are no errors, the new certificate data will be copied to the S3
    bucket specified in `my-cool-docker-compose.yml`.
1. Re-run the command from Step 3 and verify that the domains requiring
   renewal now show an expiration of 89 days.
1. Forward the certificate expiration notice, along with a confirmation that
   the certificates have been renewed, to the COOL certificates email
   distribution list so that the DevSecOps team can deploy the new
   certificates.
1. Once the certificates have been deployed by the DevSecOps team, you should
   receive an email confirmation that the deployment was successful.

Congratulations! You have successfully renewed SSL certs for multiple
domains.

## Troubleshooting tips and tricks

**Deleting certificates using certboto:** In the event an error is made while
renewing certificates, run the following command to delete the existing
certificate before renewing:

```console
docker-compose -f my-cool-docker-compose.yml run certboto delete --cert-name <name of cert>
```
