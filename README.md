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
