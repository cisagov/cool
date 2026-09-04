# Creating SSL Certificates for Assessment Mail Servers

**Purpose:** A step-by-step guide for personnel who provision mail servers
within assessment environments in the COOL.

**IMPORTANT:** You must complete this process **prior to** provisioning
(terraforming) assessment environments in the COOL that use mail servers
(i.e. RPTs, PCAs, etc.).

## Prerequisites

1. You have completed all of the prerequisite steps in
   [Creating and Renewing SSL Certificates](Creating-and-Renewing-SSL-Certificates.md).
1. You have been provided access to manage DNS records for the assessment
   email domains, wherever they are registered/hosted. Contact the DevSecOps
   team POC for access.

**NOTE:** The manual DNS challenge below is only needed for domains whose
DNS is hosted at an external registrar. If a domain's DNS is hosted in
Amazon Route 53, certbot can complete the challenge automatically — omit the
`--no-dns-route53` and `--manual` flags and skip the record-creation steps.

## Creating assessment mail server SSL certificates

1. You receive a mail server certificate request sub-ticket. Locate the domain
   name contained within the ticket — you will need this information to
   continue. When in doubt about where to find it, ask one of the COOL
   admins.
1. Open a terminal and navigate to your certboto-docker directory:
   `cd ~/code/cisagov/certboto-docker`
1. Initiate the DNS challenge process:

    ```console
    docker-compose -f my-cool-docker-compose.yml run certboto --no-dns-route53 certonly --manual --preferred-challenges dns -d <domain_name> -d *.<domain_name>
    ```

    **NOTE:** Example of a `domain_name` from the ticket: `example.com`; the
    wildcard form is `*.example.com`.

    The command above will issue the **first DNS challenge** — a string of 32
    characters. Leave the terminal at this prompt.
1. Log into the domain registrar's DNS management console using the access
   from the Prerequisites section, and locate the domain's DNS records.
1. Add a TXT record with `_acme-challenge` as the host name (this will
   always be the same host for our purposes) and the 32-character string
   generated in Step 3 as the value.
1. In your terminal, press Enter to display the **second** 32-character
   challenge.
1. Add a second `_acme-challenge` TXT record with the second challenge
   string as its value.
1. With both challenge strings entered into the registrar, and **prior to**
   pressing Enter in your terminal, test that the TXT records were
   successfully deployed by opening the Google Admin Toolbox link provided in
   your terminal in a browser.
1. If the TXT records have not yet propagated, wait until they do (click your
   browser's refresh button periodically).
1. Once you have verified that the records propagated, press Enter in your
   terminal to complete the certificate generation process. This generates
   the certificate and places it in a secure S3 bucket, which is accessed
   during the assessment environment provisioning process.
1. Annotate in the comments section of the sub-ticket from Step 1 that you
   have successfully created the certificate for the mail server, provide the
   date, and save the comment to let others working on provisioning the
   environment know that this certificate has been generated.

Congratulations! You have successfully created a mail server certificate.
