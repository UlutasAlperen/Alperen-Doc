---
title: "create-iam-user-and-group"
weight: 10
---

# Create IAM User and Group

## Create an admin-level IAM user for day-to-day account management. _This will be primary AWS login management user .

1. Navigate to the [IAM dashboard](https://console.aws.amazon.com/iam). You can also use the search bar at the top of the console to find it.
2. Click on "Users" under "Access Management" in the menu on the left, then click "Create user."
3. Give the IAM user a name like `zach-admin` (using your own name).
4. Check the box "Provide user access to the AWS Management Console."
5. Have AWS auto-generate an initial password, and leave the box checked to require a password reset on first login.
6. Create a new group for this user, called `Administrators`. Attach the `AdministratorAccess` policy to the group. Only the one policy is needed.
7. Assign the IAM user that you're creating to the new group.
8. On the "Review and create" step, if everything looks right, click "Create user."
9. Download the credentials CSV file. Note that AWS gives you a sign-in URL, the IAM username, and the initial password. _The sign-in URL has your AWS account ID in it._

--- 
## In the web console, log out of your root user, then log in as the new IAM admin user.

1. Use the sign-in URL from the credentials CSV.
2. The account ID should be pre-populated on the sign-in page, but save the ID number in case you need it for any future logins.
3. Enter the username you created and the auto-generated initial password.
4. You'll be prompted to set a new password for the IAM user. Create one and save it.
5. If prompted, set up MFA for the IAM user as well (not as critical as with the root user, but still recommended).

From this point on, use the IAM admin user unless a root user login is absolutely necessary. We'll be learning a lot more about IAM and scoped security throughout this document.

## Go back to your terminal and authenticate the AWS CLI using the IAM admin user.

1. Make sure you're using AWS CLI `2.32.0` or newer.
2. Run `aws login` and follow the browser auth flow.
3. If `aws login` is denied, your IAM user or group may be missing the permissions required for local development sign-in.

 Verify your logged-in status.
 Run:
   
```sh
aws sts get-caller-identity
```
   
The output should include your `UserId`, `Account` number, and `Arn`.
 The `Arn` should include `iam` and end with the IAM username you created.