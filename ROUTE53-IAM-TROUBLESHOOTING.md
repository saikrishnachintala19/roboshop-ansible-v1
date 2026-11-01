# Fixing Ansible Route 53 Permissions

This guide records the Route 53 permissions problem encountered while running
`roboshop.yaml`, what caused it, and how it was fixed. It is written so the
steps can be followed again without repeating the investigation.

## What this playbook does

The playbook runs on the Ansible EC2 instance, using:

```yaml
hosts: localhost
connection: local
```

It asks AWS to create or manage EC2 instances and then creates Route 53 DNS
records using the private IP addresses returned by the EC2 task.

There are two separate AWS services involved:

- **EC2** manages the virtual machines.
- **Route 53** manages the DNS records.

Permission for one service does not automatically grant permission for the
other.

## The problem we saw

The EC2 task could run, but the Route 53 task failed with an AWS
`AccessDenied` error. The error named Route 53 actions such as
`ListHostedZones` or `ListResourceRecordSets`.

The Ansible machine was using the IAM role `AnsibleEC2PracticeRole`. That role
already had `AmazonEC2FullAccess`, which permits EC2 operations, but that policy
does not grant Route 53 access.

## Why the first policy search was confusing

We created Route 53 policies directly inside the role. AWS calls these
**inline policies**. The role showed the policies as **Customer inline**.

Inline policies are already part of that role, so they do not appear in the
managed-policy attachment search. The **Attached entities: 0** value in that
view does not mean an inline policy is missing; that count is not how to check
whether an inline policy is present on the role.

To check an inline policy, open:

1. **AWS Console → IAM → Roles**
2. Select **AnsibleEC2PracticeRole**
3. Open the **Permissions** tab
4. Find and expand the Route 53 inline policy

## How we fixed it

We added Route 53 permissions to `AnsibleEC2PracticeRole`. The permissions
needed for this playbook are:

- `route53:ListHostedZones` so the Route 53 module can find the DNS zone.
- `route53:ListResourceRecordSets` so it can inspect records in the zone.
- `route53:ChangeResourceRecordSets` so it can create or update records.
- `route53:GetHostedZone` and `route53:GetChange` for zone and change status
  checks used by Route 53 operations.

The policy should limit record changes to the hosted zone used by the playbook.
Use the real hosted zone ID for `saikrishna-lab.online` in your AWS account;
the example below uses the zone ID that appeared in the permission error.

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "FindHostedZone",
      "Effect": "Allow",
      "Action": [
        "route53:ListHostedZones",
        "route53:ListHostedZonesByName"
      ],
      "Resource": "*"
    },
    {
      "Sid": "ManageRecordsInHostedZone",
      "Effect": "Allow",
      "Action": [
        "route53:GetHostedZone",
        "route53:ListResourceRecordSets",
        "route53:ChangeResourceRecordSets"
      ],
      "Resource": "arn:aws:route53:::hostedzone/Z0053784XSNYTS2952Q9"
    },
    {
      "Sid": "CheckChangeStatus",
      "Effect": "Allow",
      "Action": "route53:GetChange",
      "Resource": "arn:aws:route53:::change/*"
    }
  ]
}
```

### Add or check the inline policy

To add a policy from the role:

1. Go to **IAM → Roles → AnsibleEC2PracticeRole → Permissions**.
2. Choose **Add permissions → Create inline policy**.
3. Select the **JSON** tab and enter the policy.
4. Replace the example hosted zone ID if your zone has a different ID.
5. Continue to the policy name screen, enter a clear name such as
   `RoboshopRoute53Access`, and save.
6. Return to the role's **Permissions** tab and confirm the policy is listed.

If the policy already exists and contains the required actions and correct zone
ARN, do not create another copy. The role should keep its EC2 permissions as
well as the Route 53 permissions.

## After fixing the permissions

The Route 53 operation succeeded after the role had the required permissions.
If the playbook fails again:

1. Read the AWS action named in the error. An `AccessDenied` message usually
   identifies the missing permission.
2. Confirm that the policy is on **the role attached to the Ansible EC2
   instance**, not on a different user or role.
3. Confirm that the permission's hosted zone ARN contains the ID of the zone
   for `saikrishna-lab.online`.
4. Retry and inspect the new output. Do not paste AWS credentials or passwords
   into logs or chat.

Before rerunning the **entire** playbook, check the EC2 console for existing
instances. This playbook runs its EC2 task before its DNS task, so a failed DNS
step does not necessarily mean the EC2 step failed. Running the whole playbook
again may start additional instances and create charges. If the instances
already exist, run only the DNS task using a safe, reviewed approach rather
than blindly rerunning the EC2 task.

## Small YAML detail

The task is named `Add an AAAA record`, but its module parameter is
`type: A`. The parameter controls the DNS record type; `A` is for IPv4
addresses, which is what the private IPv4 addresses in this playbook use. The
task name is only a human-readable label and does not change the record type.
