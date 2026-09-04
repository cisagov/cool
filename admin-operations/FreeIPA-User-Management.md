# FreeIPA User Management

**Purpose:** To perform user management functions within the COOL, including
adding new users, modifying user permissions/group access, and
disabling/deleting users. COOL end-user identity (VPN, Kerberos single
sign-on, Guacamole access) is managed in FreeIPA.

FreeIPA web UIs:

- Production: `https://ipa0.cool.example.gov/`
- Staging-a: `https://ipa0.staging-a.cool.example.gov/`

## A. Prerequisites

The following must be completed before you can perform user management tasks:

1. You are assigned to perform COOL user management tasks.
1. Create a GitHub account (using your work email). The DevSecOps team uses this
   to provide you access to the COOL users script and enable pull requests.
1. Ensure that you have access to the COOL users script (in the private
   `cool-users` repository). If you do not, email the DevSecOps team for
   access.
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
       for COOL access in the ticketing system and attach your certificate
       data from the previous step as a separate attachment.
    1. Once your account is created and you receive a "Welcome to the COOL"
       email, follow the steps in the email to test your VPN connection.
1. Connect to the COOL VPN (or Staging-a VPN), test your access to the
   FreeIPA server, and verify that you have the permissions necessary to add
   new FreeIPA users.

## B. Adding COOL users

Requests to add new users (systems access requests) arrive through the
ticketing system.

### 1. Examine the ticket

Open the ticket (you will most likely need to look at the parent ticket) and
locate:

- The requestor's public key certificate.
- The name of the groups/environment(s) they require access to.
- If they are a contractor, the name of their federal lead.

**NOTE:** If the requesting user did not follow the instructions, they will
not be granted access until they provide all of the required information.

**IMPORTANT:** If a contractor is requesting access, you MUST ensure that
their federal lead has validated the need for the specific access requested.
If the federal lead cannot validate the request, you cannot proceed.

### 2. Create the new user

1. Open a browser to `https://ipa0.cool.example.gov/` (or
   `https://ipa0.staging-a.cool.example.gov/` for staging users).
1. Click the "+ Add" button in the top right corner.
1. Enter the new user's details:
    - User login: `first.last` (if a user with the exact same name already
      exists, append a number, e.g. `first.last1`, `first.last2`)
    - First name: `first`
    - Last name: `last`
1. Click "Add and Edit".
1. Under "Account Settings", change the Login shell from `/bin/sh` to
   `/bin/bash`.
1. Under "Contact Settings", enter the user's email address.
1. Click "Save" in the top left.

**If the user to be added is a federal employee:**

1. Under "Account Settings", click "Add" next to Certificate Mapping Data.
1. Click "Add" next to Certificate.
1. You will need the base64-encoded public certificate from the user's COOL
   access request (it should begin with `-----BEGIN CERTIFICATE-----`).

    **IMPORTANT:** Remove all newline characters from the certificate data so
    that it is all on a single line, then paste it into the Certificate
    field.
1. Click "Add" in the bottom right of the dialog to save your work.
1. The "Certificate mapping data" is now populated and should start with
   `X509:<i>`. Copy all of the text beginning with `X509` to the end of the
   string, which should end with `.DHS HQ`.
1. Under "Account Settings", click "Add" next to Certificate Mapping Data
   (the "Add" button underneath the text you just copied).
1. Click "Add" next to Certificate mapping data (the button above the "Add"
   button for Certificate you clicked earlier).
1. Paste the certificate mapping data text you copied into the certificate
   mapping data text box.
1. At the end of the pasted text you should see something like
   `LAST+UID=1234567689.DHS HQ`, where `LAST` is the user's last name.
   Replace the `+` with a `,` so it reads `LAST,UID=1234567689.DHS HQ`.
1. Click "Add" in the bottom right of the dialog to save your work.
1. There are now two certificate mapping data entries. Find the entry
   containing the `+` (e.g. `LAST+UID=1234567689.DHS HQ`) and click "Delete"
   to remove it. When finished, only the entry containing the `,` should
   remain.

**If the user to be added is a contractor:**

1. Under "Account Settings", click "Add" next to Certificates.
1. You will need the base64-encoded public certificate from the user's COOL
   access request (it should begin with `-----BEGIN CERTIFICATE-----`).

    **IMPORTANT:** Remove all newline characters from the certificate data so
    that it is all on a single line (a text editor such as Visual Studio Code
    can help). Paste it into the Certificate field.
1. Click "Add" in the bottom right of the dialog to save your work.
1. Click "User Groups" in the tabs at the top left of the page. Add the user
   to the `vpnusers` group and any other groups they belong in, based on the
   requirements of their role (if known at this time).

### 3. Draft the new user email

1. Using your email client, copy the canned language from the
   [New user email template](#new-user-email-template) below into a new email
   to the user, fill in their details, and send it.
1. Close out the ticket.

## C. Modifying COOL user permissions and groups

This section applies ONLY to adding existing COOL users to, or removing them
from, groups. If the user does not exist in the COOL, follow
[Adding COOL users](#b-adding-cool-users) instead.

### 1. Providing additional permissions or access to groups

1. Assign the ticket to yourself.
1. Open the ticket and determine whether the requestor is a fed or a
   contractor:
    - If a contractor, first verify their need for the additional access with
      their fed lead (a ticket submitted by the Fed Lead is usually
      sufficient).
    - If a fed, proceed to the next step.
1. Connect to the COOL VPN or COOL Staging-a VPN.
1. Access the FreeIPA server (Production or Staging-a).
1. Type the user's name in the IPA search box and click the magnifying glass
   icon.
1. Click on the user's name, then click the "User Groups" tab.
1. Click "Add".
1. In the pop-up window, checkmark the appropriate group(s) under the
   "Available" list.
1. Click the `>` button in the middle of the page to move them to the
   "Prospective" list.
1. Click "Add" at the bottom of the page to complete the process.
1. Close out the ticket.

### 2. Removing permissions or access to groups

1. On the user's group page, check the box next to the group(s) to remove
   access from, then click "Delete".
1. Close out the ticket.

## D. Deleting or disabling COOL users

Requests to delete or disable users arrive through the ticketing system.

### 1. Disabling COOL users

**NOTE:** Disable (rather than delete) a COOL user when there is a chance
they will need access again later (e.g. a special assignment, detail, or
military deployment).

1. Assign the ticket to yourself.
1. Connect to the COOL VPN or COOL Staging-a VPN.
1. Access the FreeIPA server (Production or Staging-a).
1. Type the user's name in the IPA search box and click the magnifying glass
   icon.
1. Click the "Actions" button and select "Disable" from the drop-down menu.
1. Click "OK" in the confirmation pop-up.
1. Close out the ticket.

### 2. Deleting COOL users

Delete COOL users when they are departing the organization permanently or
transitioning from a contractor to a federal employee position (the most
common cases).

1. Assign the ticket to yourself.
1. Connect to the COOL VPN or COOL Staging-a VPN.
1. Access the FreeIPA server (Production or Staging-a).
1. Type the user's name in the IPA search box and click the magnifying glass
   icon.
1. Click the "Actions" button and select "Delete" from the drop-down menu.
1. Click "OK" in the confirmation pop-up.
1. Close out the ticket.

## New user email template

> Welcome to the COOL (Cloud Optimized Operational Lab)!
>
> Congratulations — we have created your COOL account.
>
> There are only a few remaining steps you need to take to access the COOL. If you get stuck at any time, please submit a ticket.
>
> First, make sure you have satisfied all of the following prerequisites:
>
> - You are logged onto your organization-issued workstation.
> - You have access to the internet.
> - You have downloaded the latest VPN configurations from your organization's software portal.
> - You have a PIV (Personal Identity Verification) card inserted into your smart card reader.
>
> Connect to the COOL VPN:
>
> 1. Open your VPN client.
> 2. Select the COOL (or COOL STAGING-A if you have been provided Staging-a access) VPN configuration.
> 3. Follow the prompts for initial setup (i.e. enter your COOL username (firstname.lastname) for single sign-on if prompted). NOTE: Your COOL username is: \<first name.last name\>
>
> If everything goes well, your VPN client should indicate that you are connected to the COOL VPN.
>
> Test access to an assessment environment through Guacamole:
>
> 1. You currently have access to the following environments:
>     - \<Production only\> COOL Assessor Access Check (https://guac.env40.cool.example.gov)
>     - \<List other COOL environments here\>
> 2. When testing your access to COOL Assessor Access Check, you should see two Debian Linux instances which may be used to simulate accessing/switching between assessment instances in actual operational environments. Tips for using Guacamole can be found in the section immediately below.
>
> Tips for accessing Guacamole in your browser:
>
> Once connected to the COOL VPN, simply point your browser to the Guacamole environment that you have been granted access to (e.g. https://guac.env40.cool.example.gov).
>
> If there is only one instance set up in your assessment environment, you should be taken directly to that instance. If your assessment environment has multiple instances, you will see a list of available instances to connect to.
>
> Note that if you are using Chrome (recommended), and if this is your first time visiting the Guacamole URL, Chrome should prompt you about sharing your local clipboard with this site. Select "Allow" to enable seamless copy/paste functionality between your laptop's host OS and the remote instances that you access with Guacamole.
>
> To connect to a remote system, locate its name in the list under "ALL CONNECTIONS" and click on it. Note that some connections will allow you to control the cursor on the remote instance, while other connections may be "View Only".
>
> While connected to a remote system, you should be able to use the system as you would any other remote desktop. Refer to the Guacamole user documentation for full details. Some useful features of Guacamole are:
>
> - Viewing/hiding the Guacamole menu while connected to a remote system.
> - Copying/pasting text between your local system and the remote system.
> - Transferring files between your local system and the remote system (including drag and drop copying from your local system to the remote system, but not vice versa). Files copied to the remote system can be found in the /home/vnc/Documents directory.
>
> To disconnect from a remote system, press CTRL-OPT-SHIFT (or CTRL-ALT-SHIFT) to bring up the Guacamole menu, then select the dropdown menu labeled guacuser in the upper-right corner and select "Disconnect". At that point, you will have the option of returning to the Guacamole home screen if you want to log on to a different remote system.
>
> Note that selecting "Logout" from Guacamole will appear to have no effect since your Kerberos credentials will continue to keep you logged in. When you are done using Guacamole, you can simply close the Guacamole browser window.
>
> Thanks,
>
> The COOL Team

## Appendix: Troubleshooting tips and tricks

Recommended actions for resolving COOL user access issues:

1. **User is unable to connect to the COOL VPN:**
    1. Check that the user has the latest VPN configurations by having them
       check in with device management or download the package from your
       organization's software portal.
    1. If they are running the latest configuration, check the VPN
       connection's networking settings in the client (Preferences → select
       connection → Edit → Networking tab). If there is a route in the
       routing table, delete it, restart the VPN client, and try to connect
       again.
1. **User is connected to the COOL VPN but cannot access their assigned
   Guacamole environments:** Ask the user if they have disconnected and
   reconnected to the COOL VPN within 24 hours. If not, their Kerberos
   credentials have likely expired — have them disconnect and reconnect to
   the COOL VPN.
