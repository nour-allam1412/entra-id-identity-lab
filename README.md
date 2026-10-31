# Microsoft Entra ID Identity Lab

A hands-on lab that sets up identity and access management for a simulated company in Microsoft Entra ID: bulk user onboarding, department security groups, least-privilege admin roles, multifactor authentication (MFA) and passkey sign-in, and self-service password reset (SSPR). Every result below was checked in the Entra admin center logs or error messages.

**Author:** Nour Allam | **Date:** October 2026 | **Tools:** Microsoft Entra admin center, Microsoft Authenticator, CSV

## At a glance

| Part | What I did | Result |
|---|---|---|
| 1. Bulk onboarding | Created 25 users from a CSV file and organized them into three department security groups | 25 accounts, each created exactly once; groups of IT 9, Sales 9, HR 7 |
| 2. Least privilege | Assigned Helpdesk Administrator and User Administrator roles and tested the boundary between them | Helpdesk Administrator could reset a standard user's password but was blocked from resetting a User Administrator's |
| 3. MFA and passkeys | Checked security defaults, registered Authenticator and a passkey, and analyzed the sign-in logs | Every sign-in required MFA; the passkey sign-in succeeded with no password step |
| 4. Self-service password reset | Tested a password reset as a standard user | Reset succeeded and was confirmed in the audit log |

## Scenario

"Contoso Labs" is a fictional company with three departments (IT, Sales, HR). As the IT administrator I had to:

1. Onboard employees quickly and consistently.
2. Give the help desk only the permissions it needs.
3. Make sure sign-ins are protected with multifactor authentication.
4. Let users manage their own passwords to reduce help desk tickets.

## Environment

| Item | Detail |
|---|---|
| Platform | Microsoft Entra ID (free tier) |
| Tenant owner | A personal Microsoft account added as an external (`#EXT#`) user |
| Users | 25 fictional cloud-only users created from a CSV file |
| Group members | 25 in total: IT 9, Sales 9, HR 7 |
| Test devices | One phone with Microsoft Authenticator; browser private windows for signing in as test users |

---

## Part 1: Bulk user onboarding and groups

**Goal:** create users in bulk instead of one by one, and organize them by department.

**Steps**
1. Downloaded the CSV template from **Users > Bulk operations > Bulk create**.
2. Filled in fictional users (display name, user name, initial password, job title, department, location and contact fields), leaving the template's header row unchanged.
3. Uploaded the file and checked the bulk operation results.
4. Created one security group per department and added the members.

**Result**

| Group | Members |
|---|---|
| IT | 9 |
| Sales | 9 |
| HR | 7 |
| **Total** | **25** |

**Bulk upload outcome:** the first upload failed for all 25 rows because the CSV still contained a placeholder domain ("The domain portion of the userPrincipalName property is invalid. You must use one of the verified domain names in your organization."). After correcting the domain, I submitted the file again, and then once more while the earlier job was still processing. The two jobs ran concurrently: each of the 25 accounts was created exactly once, and the other job reported "Another object with the same value for property userPrincipalName already exists." One job reported 21 failures and 4 successes, the other 21 successes and 4 failures, and the 4 accounts were the same in both. The audit log agrees: every account has one successful "Add user" entry and one "ObjectAlreadyExists" failure within about five seconds. The four accounts reported as failed in the last results file existed anyway, so I added them to the HR group manually.

**Lesson:** wait for a bulk operation to finish, and check the directory, before re-uploading.

The sample file in `/data` masks the passwords and uses a generic domain.

---

## Part 2: Roles and least privilege

**Goal:** give the help desk only the access it needs, and prove the boundary.

**Roles assigned**

| User | Role |
|---|---|
| Lucas Bennett | Helpdesk Administrator |
| Oliver Reeves | User Administrator |

**Tests (signed in as Lucas, in a private window)**

| Test | Result |
|---|---|
| Reset the password of a standard user (Emma Brooks) | **Allowed**: a temporary password was issued and used to sign in as that user |
| Reset the password of a User Administrator (Oliver Reeves) | **Blocked**: "The password can not be reset. This may be due to an incorrect level of administrative privilege or if trying to reset your own password." |
| Reset the password of the tenant owner account | **Blocked**, for a different reason (see below) |

**Finding:** the Helpdesk Administrator role can reset a standard user's password but not a User Administrator's, which confirms the role boundary. I confirmed I was signed in as Lucas, so the "own password" cause of the error on Oliver did not apply.

**What broke:** resetting the tenant owner's password returned "...is a Microsoft account that is managed by the user. Only [the user] can reset their password." The owner is a personal Microsoft account added as an external user (`#EXT#` in the user name). Entra cannot reset consumer Microsoft account passwords, so this block says nothing about role permissions. I switched the test to a cloud-only administrator (Oliver) to validate the role boundary properly.

---

## Part 3: MFA and sign-in security

**Goal:** verify that sign-ins are protected by multifactor authentication.

**Steps**
1. Opened **Manage security defaults** and found it already **Enabled** for the tenant. I did not enable it myself.
2. Registered Lucas Bennett with **Microsoft Authenticator** and a **passkey**.
3. Signed in several times and reviewed **Monitoring > Sign-in logs**.

**Sign-in log results (Lucas Bennett, Azure Portal, 10/30/2026)**

| Time | Status | Authentication requirement |
|---|---|---|
| 1:08:26 AM | Interrupted (error 50140) | Multifactor authentication |
| 1:08:30 AM | Success | Multifactor authentication |
| 1:59:34 AM | Success | Multifactor authentication |

**Authentication Details for those sign-ins**

| Time | Step | Detail |
|---|---|---|
| 1:08:26 AM | Password (first factor) | "Password in the cloud": correct password |
| 1:08:26 AM | MFA | "MFA completed in Azure AD" (the one interactive MFA step) |
| 1:08:30 AM | MFA | "Previously satisfied": satisfied by a claim in the token |
| 1:59:31 AM | Password (first factor) | Correct password |
| 1:59:31 / 1:59:34 AM | MFA | "Previously satisfied": satisfied by a claim in the token |
| 2:07:50 AM | First factor and MFA | Both "Previously satisfied" by claims in the token |

**Passkey sign-in test (passwordless)**

| Time | Step | Detail |
|---|---|---|
| 3:20:51 AM | MFA | "Previously satisfied" by a claim in the token |
| 3:23:49 AM | Passkey (synced) | Success; no password step logged |
| 3:23:49 AM | MFA | "Previously satisfied": satisfied by a multi-factor device |
| 3:23:51 AM | MFA | "Previously satisfied" by a claim in the token |

**Findings**
- Every sign-in required **multifactor authentication**.
- MFA was performed once; later sign-ins reused the MFA claim already in the session token instead of prompting again. This is expected behavior, not a bypass.
- **Conditional Access: Not Applied.** Security defaults are not a Conditional Access policy. Conditional Access needs Entra ID P1/P2 and was not used in this lab.
- **Error 50140** looked like a failure but is the normal "Keep me signed in?" prompt, which Entra logs as an interruption. The message in the log and Microsoft's documentation confirm this.
- In the first sign-ins, the passkey was registered but not used: the first factor was the password each time it was prompted.
- In the passkey test, sign-in succeeded and **no password step appears**. The log credits the MFA requirement to a **multi-factor device**, which fits a passkey (the device you hold plus a PIN or biometric).
- The credential is labeled **"synced"**, meaning it can be backed up and available across devices through a password manager or platform account, as opposed to being tied to a single hardware key.

---

## Part 4: Self-service password reset

**Goal:** let a standard user recover their own password without a help desk ticket.

**Test:** a standard user (Emma Brooks, no admin role) completed a self-service reset, and I checked **Monitoring > Audit logs**.

| Field | Value |
|---|---|
| Service | Self-service Password Management |
| Activity | Reset password (self-service) |
| Status | Success ("Successfully completed reset.") |
| Target user | Emma Brooks |
| Date | 10/30/2026 |

**Findings**
- A standard cloud-only user completed a self-service reset in this tenant.
- Documentation I found on free-tier SSPR was inconsistent, so I relied on the log as evidence. I did not determine which tenant setting or license enabled it.
- My own admin account is a personal Microsoft account, whose password is managed outside Entra (see Part 2).

---

## What broke and how I fixed it

| Problem | Cause | Resolution |
|---|---|---|
| CSV upload failed for all rows | A placeholder domain was left in the user names | Replaced it with the tenant's verified domain and re-uploaded |
| Bulk upload reported "userPrincipalName already exists" for some rows | Re-uploaded while the earlier job was still running, so two jobs created the same accounts concurrently | Compared the results files and audit log, confirmed every account was created exactly once, and added the 4 accounts reported as failed to their group |
| Password reset of the owner account failed | Personal Microsoft account added as an external user | Tested role boundaries on a cloud-only administrator instead |
| Sign-in log showed "Interrupted" | Normal "Keep me signed in?" prompt (error 50140) | Verified in the log message and documentation; no action needed |
| Free-tier SSPR documentation conflicted | Inconsistent sources | Used the audit log as evidence |

## Security notes

- All users, names, addresses and phone numbers are fictional. The `+1 555-01xx` phone numbers are reserved for fiction.
- The lab accounts were created with a shared placeholder password, which is not published; the sample file masks it. Passwords on some accounts were changed during testing, and the rest still use the placeholder.
- The test tenant is separate from any production environment.

## Skills demonstrated

Microsoft Entra ID administration, bulk user provisioning (CSV), security groups, role-based access control and least privilege, MFA and passkey configuration, log analysis (sign-in and audit logs), troubleshooting, and documentation.

## Repository contents

```
/README.md                      this file
/data/users-sample.csv          sample user file (passwords masked)
```
