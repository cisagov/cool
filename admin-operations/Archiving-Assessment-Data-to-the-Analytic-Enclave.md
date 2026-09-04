# Archiving Assessment Data to the Analytic Enclave

**Purpose:** To send archived data from FAST, RPT, RTA, and RVA assessment
environments to the Analytic Enclave (AE) S3 bucket at the end of an
assessment.

**About the archive script:** The script is
`archive-artifact-data-to-bucket.sh` and it is located in the `/home/vnc`
directory:

- On the **Kali** instances for RPT and RVA environments.
- On the **terraformer** instance for FAST and RTA environments (these
  environments have a sole terraformer provisioned).

The script creates a gzipped tar archive of the directory containing the
assessment artifacts and copies that archive to the appropriate S3 bucket.

## Steps

1. Receive a ticket entitled "Awaiting Data Archive and Report."
1. Connect to the COOL VPN and access the assessment environment using
   Guacamole (i.e. `https://guac.env<X>.cool.example.gov`).
1. Open a shell on the appropriate instance and navigate to `/home/vnc`:
    - For RPTs and RVAs: one of the Kali instances.
    - For FASTs and RTAs: the terraformer instance.
1. Run the archive script, specifying the correct path to the artifacts
   directory:
    - For FASTs, RPTs, and RTAs:

        ```console
        sudo bash archive-artifact-data-to-bucket.sh ../../share/
        ```

    - For RVAs:

        ```console
        bash archive-artifact-data-to-bucket.sh /share/Archive
        ```

1. When prompted to continue the archiving process, type `y`.

    **NOTE:** This process may take a few minutes. Once the upload to the S3
    bucket is complete you will receive a confirmation on the command line.

1. Confirm successful archiving in the "Awaiting Data Archive and Report"
   ticket's comments section, then close the ticket out.

## Troubleshooting tips

- **Archiving not completing / errors?** Try running the script on a
  different Kali instance.
- **Permissions error?** Run the command with `sudo` if you haven't already.
- **"Path not found" error?** Make sure you have the correct path to the
  assessment artifacts folder. When in doubt, ask another team member.
