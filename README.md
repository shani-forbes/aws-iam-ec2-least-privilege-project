# AWS IAM & EC2 Least-Privilege Access Project

## Overview

This project demonstrates the design and validation of a least-privilege access model in AWS using IAM, EC2, and S3.

I built the environment around a simulated small-company scenario that required different levels of access for employees while also allowing an application running on EC2 to securely retrieve a specific file from S3.

The project focuses not only on configuring AWS permissions, but also on evaluating access requirements, separating human and workload identities, troubleshooting authorization issues, and validating that both allowed and denied actions behaved as expected.

## Scenario

A small company is setting up its AWS environment and needs to provide access based on different job responsibilities.

Three employees require AWS access:

- **Maya – Cloud Administrator:** Responsible for managing the company's AWS environment and requires administrative access.
- **Andre – Developer:** Works with the company's EC2 infrastructure and requires access to manage EC2 resources.
- **Nia – Finance:** Needs visibility into AWS billing and cost information but should not be able to modify billing configurations.

The company also runs an application on an EC2 instance. The application needs to retrieve a specific file stored in an S3 bucket.

For security reasons, the application should:

- Access only the required S3 object.
- Be able to read the object but not upload or modify objects.
- Use an IAM role with temporary credentials rather than long-term access keys stored on the EC2 instance.

The goal was to design an access model that satisfied these requirements while following the principle of least privilege.

## Project Objectives

For this project, I wanted to:

- Design role-based AWS access for users with different responsibilities.
- Apply least privilege when deciding what each user or workload actually needed.
- Configure an EC2 instance to securely retrieve a specific file from S3 using an IAM role rather than stored access keys.
- Test the configuration to confirm that intended actions were allowed and unintended actions were denied.

## IAM User and Group Design

Based on the scenario, I created three IAM groups to manage access according to each employee's responsibilities:

| User | Role | IAM Group | AWS Managed Policy |
|------|------|-----------|--------------------|
| Maya | Cloud Administrator | Cloud_Admin | AdministratorAccess |
| Andre | Developer | Developers | AmazonEC2FullAccess |
| Nia | Finance | Finance | AWSBillingReadOnlyAccess |

I managed permissions through IAM groups rather than attaching policies directly to individual users, making access easier to manage as users are added or responsibilities change.

### Reviewing Finance Access

My initial choice for the Finance group was AWS's `Billing` managed policy. The name appeared to match Nia's responsibilities, but after reviewing the permissions included in the policy, I noticed that it provided more than visibility into billing information.

The policy included write permissions across several billing-related services. Since Nia's requirement was to monitor costs rather than make billing changes, I decided this was broader access than necessary.

I replaced it with `AWSBillingReadOnlyAccess`, which better matched the Finance role's requirements while reducing unnecessary permissions.

## EC2 Workload Access to S3

The next requirement was to allow an application running on EC2 to retrieve a specific file from an S3 bucket.

Rather than creating an IAM user and storing long-term access keys on the instance, I used an IAM role for EC2. This allows the instance to receive temporary AWS credentials automatically and keeps the workload's permissions separate from the permissions assigned to human users.

### Creating the S3 Access Policy

I created a custom policy, `EC2-S3-TestFile-ReadOnly`, that allowed only the following action:

- `s3:GetObject`

The permission was scoped to a single object, `test-file.txt`, rather than granting access to the entire S3 bucket.

This meant the EC2 workload could retrieve the file it needed without receiving broader read or write access to S3.

### Creating the EC2 Role

I created the `EC2-S3-TestFile-Role` and configured EC2 as its trusted service.

The role combined two separate controls:

- **Trust policy:** allowed the EC2 service to assume the role.
- **Permissions policy:** defined what the instance could do after assuming it — in this case, retrieve the specified S3 object.

I then attached the role to the EC2 instance.

## Testing, Troubleshooting, and Validation

With the EC2 instance running and the IAM role attached, I connected to the instance to test whether the access model worked as intended.

> **Note:** AWS account-specific identifiers and infrastructure details have been redacted from this documentation.

### Connecting to the EC2 Instance

My initial attempt to connect using EC2 Instance Connect from the AWS console was unsuccessful. I switched to connecting from my local terminal using SSH and the key pair created when launching the instance.

After restricting the private key's permissions, I connected successfully using the Amazon Linux `ec2-user`.

```bash
chmod 400 "EC2 for Iam.pem"

ssh -i "EC2 for Iam.pem" ec2-user@<EC2-PUBLIC-DNS>
```

### Verifying the Instance Identity

Before testing S3 access, I verified which AWS identity the EC2 instance was using:

```bash
aws sts get-caller-identity
```

The returned ARN showed that the instance was operating as an assumed role session for `EC2-S3-TestFile-Role`.

This confirmed that the workload was using the EC2 IAM role rather than credentials belonging to one of the human IAM users.

### Troubleshooting S3 Access

My first attempt to retrieve `test-file.txt` resulted in an `AccessDenied` error.

I reviewed the custom IAM policy and compared its resource ARN with the actual S3 bucket and object. I found that I had incorrectly specified the resource in the policy, so the permission did not apply to the object I was trying to retrieve.

I corrected the policy to reference the exact object:

```text
arn:aws:s3:::iam-ec2-project-<ACCOUNT-ID>-us-east-1-an/test-file.txt
```

After correcting the resource ARN, I repeated the test.

### Validating Read Access

I used the S3 API to retrieve the object:

```bash
aws s3api get-object \
  --bucket iam-ec2-project-<ACCOUNT-ID>-us-east-1-an \
  --key test-file.txt \
  downloaded-test-file.txt
```

The request succeeded. I then verified the downloaded file:

```bash
cat downloaded-test-file.txt
```

The expected contents were returned:

```text
Hello from my AWS IAM + EC2 project!
```

This confirmed that the EC2 role could retrieve the specific S3 object defined in the policy.

### Validating That Write Access Was Denied

A successful read confirmed what the role could do, but I also wanted to verify what it could not do.

I created a local test file on the EC2 instance and attempted to upload it to the S3 bucket:

```bash
echo "EC2 should NOT be allowed to upload this" > unauthorized-upload.txt

aws s3api put-object \
  --bucket iam-ec2-project-<ACCOUNT-ID>-us-east-1-an \
  --key unauthorized-upload.txt \
  --body unauthorized-upload.txt
```

AWS returned `AccessDenied` for `s3:PutObject`.

This was the expected result because the role's policy allowed `s3:GetObject` but did not grant `s3:PutObject`. Together, the successful read and denied write confirmed that the role was operating within the intended permissions.
