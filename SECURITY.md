# Security Policy

## Disclaimer

This repository contains sample code for demonstration and learning. It is **not production-ready**.
Review and test it against your own security, compliance and operational requirements before any
production use. The authors and AWS make no warranty regarding its fitness for production workloads.

## Reporting Vulnerabilities

If you discover a potential security issue in this project we ask that you notify AWS/Amazon Security
via our [vulnerability reporting page](https://aws.amazon.com/security/vulnerability-reporting/).
Please do **not** create a public GitHub issue.

## AWS Services Used

AWS Backup (read-only `ListBackupJobs`), AWS Organizations (organization ID condition), Amazon EventBridge,
AWS Lambda (Python 3.13), AWS Security Token Service (AWS STS), AWS Identity and Access Management (IAM),
Amazon S3, AWS KMS, Amazon CloudWatch Logs, AWS Systems Manager Parameter Store and AWS CloudFormation.

## Prerequisites and Permissions

Deploying the solution requires permissions to create named IAM roles and managed policies
(`CAPABILITY_NAMED_IAM`) in two accounts:

- **Reporting account** (`template-reporting.yml`): an S3 bucket and bucket policy, a KMS key, a CloudWatch
  Logs log group, a Lambda function, an EventBridge rule and an IAM role and policy.
- **Management account** (`template-management.yml`): an IAM role and policy and an SSM parameter.

AWS Backup cross-account monitoring must be turned on in the management account.

The solution is **read-only** against AWS Backup. The management account role allows only
`backup:ListBackupJobs`. That action does not support resource-level permissions, so it uses
`Resource: "*"`. The role trusts only the Lambda execution role in the reporting account
(`Backup-Report-Role`) and only for principals that belong to your organization
(`aws:PrincipalOrgID`). The Lambda role can assume only the management account role, write objects under
`BackupReports/` in the report bucket, and write to its own log group.

## Known Security Considerations

The following items were reviewed and accepted for this sample. Revisit them before any production use.

| Item | Why it is acceptable here | Production recommendation |
|------|---------------------------|---------------------------|
| Lambda function is not in a VPC | It calls only AWS APIs over TLS and exposes no listener | Add `VpcConfig` with private subnets and VPC endpoints for AWS STS, AWS Backup and Amazon S3 |
| No dead-letter queue | The function runs on a schedule, and the next run regenerates the report for the day | Add `DeadLetterConfig` and an alarm on the `Errors` metric of the function |
| S3 Object Lock is not enabled | Object Lock would block the clean-up steps for this sample | Enable Object Lock if you need to keep reports immutable |
| S3 buckets use SSE-S3 rather than a customer managed key | Reports contain job metadata, not application data | Use `SSEAlgorithm: aws:kms` with a customer managed key if your policy requires it |
| S3 server access logs are written to the report bucket itself (`access-logs/`) | Keeps the sample to one bucket | Send access logs to a separate log bucket with its own retention |
| `kms:*` to the account root in the key policy | Standard default that keeps the key manageable if stack roles are removed | Scope key administration to a named deployment role |
| Report bucket and KMS key are deleted with the stack | Clean-up is expected for a sample (the bucket must be emptied first) | Set `DeletionPolicy: Retain` on the bucket and key if you must keep reports and logs |
| Fixed role names (`Backup-Report-Role`, `Backup-Report-Policy`) | The management template trusts the reporting role by name | Parameterize the names if you need more than one deployment per account |

## Operational Notes

- Reports contain account IDs, resource ARNs, vault names, backup plan IDs and job status for every account in
  your organization. Treat the report bucket as sensitive. Grant read access only to the teams that need it,
  and do not share report files outside your organization.
- Anyone who can change the Lambda function code or its execution role in the reporting account can read
  backup job metadata for the whole organization. Restrict `lambda:UpdateFunctionCode`,
  `lambda:UpdateFunctionConfiguration` and `iam:PassRole` for this function.
- The trust policy on the management account role stores the unique ID of the reporting Lambda role. If you
  delete and recreate the reporting stack, redeploy the management stack.
- The default schedule runs once a day at 23:59 UTC. A shorter schedule (for testing) increases Lambda, S3 and
  AWS Backup API usage. Set it back when you finish testing.

## Resource Cleanup

1. In the reporting account, empty the report bucket, including all object versions and delete markers
   (the bucket has versioning turned on and holds the access logs).
2. Delete the stack that you created from `template-management.yml` in the management account.
3. Delete the stack that you created from `template-reporting.yml` in the reporting account. The KMS key
   goes into a pending deletion period and is removed automatically.
4. If you turned on AWS Backup cross-account monitoring only for this sample, turn it off. AWS Backup keeps
   aggregated job data for 30 days after you turn it off.

## Dependencies

The Lambda function uses only the Python standard library and the AWS SDK for Python (Boto3) that is
provided by the Lambda runtime. There are no third-party packages to patch.
