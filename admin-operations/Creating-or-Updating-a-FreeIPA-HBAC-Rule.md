# Creating or Updating a FreeIPA HBAC Rule

These steps are performed as part of the post-apply process when building an
assessment environment (see
[Provisioning Assessment Environments](Provisioning-Assessment-Environments.md)).

Existing environments (currently 0–100 in Production and 0–29 in Staging-a)
already have an HBAC (host-based access control) rule, and do not need a new
rule created. Instead, the existing rule will generally be **updated**.

## Steps

1. Browse to the COOL FreeIPA HBAC Rule page ("Policy" menu in the FreeIPA
   web UI, Production or Staging-a).
1. Find the rule name under the "Policy" menu, which will be labeled
   `env<X>_production_access` or `env<X>_staging-a_access`.
1. Under the "Hosts" list of the "Accessing" section, click Add.
1. In the list, locate the environment that corresponds with the same number
   as the HBAC rule in the "Available" column, click the `>` arrow to move it
   to the "Prospective" column, and click Add.
1. This saves automatically, so the rule is now updated successfully.
1. Test the access by navigating to the environment (i.e.
   `https://guac.env<X>.cool.example.gov` or
   `https://guac.env<X>.staging-a.cool.example.gov`).

**IMPORTANT:** HBAC rules should NEVER be deleted from FreeIPA, including
during environment teardown (see
[Deleting Assessment Environments](Deleting-Assessment-Environments.md)).
