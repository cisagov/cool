# FreeIPA Group Management

**Purpose:** To create or remove user groups in COOL Production or COOL
Staging-a using FreeIPA.

**IMPORTANT — the two types of groups in the COOL:**

- **User Groups** — collections of users usually associated with performing
  like functions within the COOL, such as operators assigned to the same
  assessment or systems administrators performing maintenance and management
  functions. Assessment user groups are usually in the format `VMAxxxxxxx`.
- **Assessment Environment Groups** — the containers in which particular
  assessment assets (i.e. RVA or RPT virtual instances) are provisioned and
  then de-provisioned. These appear in FreeIPA in the format
  `env<XY>_production` or `env<XY>_staging-a`. The user group associated with
  the assessment (`VMAxxxxxxx`) provides access to the assessment environment
  group for the duration of the assessment.

## A. Prerequisites

The following must be completed before you can perform group management
tasks.

**NOTE:** Group management tasks generally go hand-in-hand with COOL user
management (creating/deleting users) and provisioning/destroying COOL
environments.

1. You are assigned to perform COOL user management tasks.
1. Create a GitHub account using your work email. The DevSecOps team uses this
   to provide you access to the `cisagov` repositories.
1. Ensure that you have access to the `cisagov` repositories. If you do not,
   email the DevSecOps team for access.
1. Install the latest VPN client configuration for the COOL VPN or the COOL
   Staging-a VPN from your organization's software portal.
1. Perform the COOL user enrollment for yourself:
    1. Insert your PIV into your smart card reader.
    1. Open Terminal and run the following command to copy your public key to
       your clipboard (you will not see any output):

        ```console
        pkcs15-tool --read-certificate 1 | pbcopy
        ```

        **NOTE:** `pbcopy` is the macOS clipboard helper; on Linux, pipe to
        `xclip -selection clipboard` (X11) or `wl-copy` (Wayland) instead.

    1. Create (or have your Fed Lead create) a system access request ticket
       for COOL access and attach your certificate data from the previous
       step as a separate attachment.
    1. Once your account is created and you receive a "Welcome to the COOL"
       email, follow the steps in the email to test your VPN connection.
1. Connect to the COOL VPN (or Staging-a VPN), test your access to the
   FreeIPA server (Production: `https://ipa0.cool.example.gov/`, Staging-a:
   `https://ipa0.staging-a.cool.example.gov/`), and verify that you have
   the permissions necessary to manage FreeIPA groups.

## B. Creating new user groups

1. You will receive "assign operators" ticket requests.
    1. Open the ticket and examine it.
    1. In your browser, open the COOL environment tracking spreadsheet, click
       on the accounts tab for the appropriate COOL instance (Production or
       Staging-a), and locate an assessment environment whose "Current Uses"
       column is labeled "AVAILABLE FOR USE". Then, in the "Current Uses"
       column, fill in:
       `<type of assessment> - <unique assessment ID> <assessment federal lead name> "COMING SOON"`
       and ensure the box is color-coded appropriately (beige).
1. Connect to the COOL (or Staging-a) VPN.
1. Navigate to the COOL FreeIPA console (Production or Staging-a).
1. Click on "Groups" and then click "Add".
1. In the pop-up window, fill in the name of the new group (the unique
   assessment ID from the ticket, usually in the format `VMAxxxxxxx`) and the
   description (copy from an existing assessment of the same type), then
   click "Add and Edit".
1. Click "Add" to add users.
1. Select the users to add to the group by checking the box next to their
   username and clicking the left-pointing arrow.
1. Once done, click "Add".
1. **IMPORTANT:** Click on the "User Groups" tab on the **left** side of the
   FreeIPA GUI (denoted by "xx group members") to add the Feds group for the
   specific assessment type (i.e. `rva_fed_leads` for RVAs or
   `rpt_fed_leads` for RPTs).

    **NOTE:** The Feds group must always be assigned to an assessment in
    addition to the operators.
1. Click on the "User Groups" tab on the **right** side of the FreeIPA GUI
   (denoted by "xx is a member of") to begin adding the new user group to the
   assessment environment.
1. Click "Add" to display the list of assessment environments.
1. Select the assessment environment to add the user group to by checking the
   box next to the environment (the environment you selected in Step 1b) and
   clicking the left-pointing arrow.
1. Once done, click "Add".

You have successfully created a group and added users.

## C. Removing user groups

**IMPORTANT:** Removing user groups is normally only done at the end of an
assessment after the data archival process has been completed, OR if an
assessment is cancelled. You will only remove **user groups** (usually in the
format `VMAxxxxxxx`) unless otherwise directed. **NEVER** remove environment
groups (format `env<XY>_production` or `env<XY>_staging-a`) unless explicitly
directed by DevSecOps team personnel.

1. You will receive ticket requests to destroy an environment (for which
   removing the user group is a step).
    1. Open the ticket and examine it.
    1. In your browser, open the COOL environment tracking spreadsheet, click
       on the accounts tab for the appropriate COOL instance (Production or
       Staging-a), and locate the assessment environment whose "Current Uses"
       column is labeled with the assessment group to be removed.
1. Connect to the COOL (or Staging-a) VPN.
1. Navigate to the COOL FreeIPA console (Production or Staging-a).
1. Click on "Groups", enter the name of the assessment (usually in the format
   `VMAxxxxxxx`) into the search box, and click the magnifying glass icon.
1. Check the box to the left of the assessment group to be removed, then
   click "Delete".

You have successfully removed the assessment group from the COOL.
