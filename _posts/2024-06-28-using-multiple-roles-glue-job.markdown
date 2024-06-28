---
layout: post
title: Assuming different roles within Glue Job to perform actions scoped within different roles
date: 2024-06-28 10:00:00 -0500
description: How to assume different roles within Glue Job to perform tasks that are not allowed by the Glue IAM Role assigned, but available thourhg a different role that can be assumed by the Glue IAM Role.
img: posts_imgs/glue-assume-role/glue_assume_role.png
tags: [python, AWS, Glue, IAM, job]
---

Currently, AWS Glue jobs only supports a single IAM role. In some scenarios, you may want to assume another role with different scoped permissions than the Glue Job IAM Role. In these cases, one common workaround is to set the Glue IAM role as a trusted identity in the role with different scoped permissions and assume the role within the Glue Job script and perform the operations allowed only for the assumed role.

The Glue IAM roles is defined in the "Job details" tab within the Glue Job.
![Glue_job_details](/assets/img/posts_imgs/glue-assume-role/glue_job_details.png)

For this exercise we will create a Python Shell job and we will provide a Glue Role with S3 permissions only for *aws-glue* buckets (which are bucket created by default by Glue to allocate the script). The Glue Role will be named "glue-role-test". The policy for this will be as follows:

```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Action": [
                "glue:*",
                "s3:GetBucketLocation",
                "s3:ListBucket",
                "s3:ListAllMyBuckets",
                "s3:GetBucketAcl",
                "ec2:DescribeVpcEndpoints",
                "ec2:DescribeRouteTables",
                "ec2:CreateNetworkInterface",
                "ec2:DeleteNetworkInterface",
                "ec2:DescribeNetworkInterfaces",
                "ec2:DescribeSecurityGroups",
                "ec2:DescribeSubnets",
                "ec2:DescribeVpcAttribute",
                "iam:ListRolePolicies",
                "iam:GetRole",
                "iam:GetRolePolicy",
                "cloudwatch:PutMetricData"
            ],
            "Resource": [
                "*"
            ]
        },
        {
            "Effect": "Allow",
            "Action": [
                "s3:CreateBucket"
            ],
            "Resource": [
                "arn:aws:s3:::aws-glue-*"
            ]
        },
        {
            "Effect": "Allow",
            "Action": [
                "s3:GetObject",
                "s3:PutObject",
                "s3:DeleteObject"
            ],
            "Resource": [
                "arn:aws:s3:::aws-glue-*/*",
                "arn:aws:s3:::*/*aws-glue-*/*"
            ]
        },
        {
            "Effect": "Allow",
            "Action": [
                "s3:GetObject"
            ],
            "Resource": [
                "arn:aws:s3:::crawler-public*",
                "arn:aws:s3:::aws-glue-*"
            ]
        },
        {
            "Effect": "Allow",
            "Action": [
                "logs:CreateLogGroup",
                "logs:CreateLogStream",
                "logs:PutLogEvents"
            ],
            "Resource": [
                "arn:aws:logs:*:*:*:/aws-glue/*"
            ]
        },
        {
            "Effect": "Allow",
            "Action": [
                "ec2:CreateTags",
                "ec2:DeleteTags"
            ],
            "Condition": {
                "ForAllValues:StringEquals": {
                    "aws:TagKeys": [
                        "aws-glue-service-resource"
                    ]
                }
            },
            "Resource": [
                "arn:aws:ec2:*:*:network-interface/*",
                "arn:aws:ec2:*:*:security-group/*",
                "arn:aws:ec2:*:*:instance/*"
            ]
        }
    ]
}
```

Since this role will used directly by the Glue job we need to set the "Trust Relationships" of the role to be assume by the Glue service. 

```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Principal": {
                "Service": "glue.amazonaws.com"
            },
            "Action": "sts:AssumeRole"
        }
    ]
}
```

Now this role can be set for any Glue job and will have permissions to read and write in some specific S3 buckets, send data to cloudwatch etc.

Now let's create another IAM role with enough permissions to read S3 data from a different bucket. This role can be in the same account or in a different account if you wish to allow the Glue Job to access S3 bucket in a different account. This IAM role will be name "glue-role-test-to-be-assumed". For the permission policies, we can use a AWS managed policy with full permissions for S3 "AmazonS3FullAccess". For the "Trust relationships" tab it is very important to allow the role to be assumed by another role.

Trust relationship policy:

```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Sid": "Statement1",
            "Effect": "Allow",
            "Principal": {
                "AWS": "arn:aws:iam::<ACCOUNT_ID_WEHERE_GLUE_JOB_RUNS>:role/glue-role-test"
            },
            "Action": "sts:AssumeRole"
        }
    ]
}
```

You can see that the entity to allow to assume the role "arn:aws:iam::<ACCOUNT_ID_WEHERE_GLUE_JOB_RUNS>:role/glue-role-test" is using the account id where the Glue job will be running (for this exercise is the same account, but you can use it in cross account scenarios) and the name of the role that will be assuming this new role, in this case we named it "glue-role-test".

Here is an example of the process to assume role to access another account resources.
![Glue_job_details](/assets/img/posts_imgs/glue-assume-role/role_assumed.png)

In the Glue job, we will be using the following script:

```python
import boto3
import glob
s3 = boto3.client('s3')
response = s3.list_buckets()

print(glob.glob("/tmp/*")) # This line shows the local files in the Glue node. We will try to download a file from S3 into this local folder.

# Output the bucket names using the Glue role. Glue role has permissions to list all buckets
print('Existing buckets:')
for bucket in response['Buckets']:
    print(f'  {bucket["Name"]}')

# These 2 lines does not work because the Glue Role "glue-role-test" has no permissions to read from S3 bucket "bucket_kjshadjas_3432_dsadas". You can try uncomment these lines and test.
#s3 = boto3.client('s3')
#s3.download_file('bucket_kjshadjas_3432_dsadas', 'test_path/some_demo_file.gz', '/tmp/file.gz')

target_role_arn="arn:aws:iam::ACCOUNT_ID_WHERE_NEW_ROLE_TO_ASSUME_EXISTS:role/glue-role-test-to-be-assumed"

sts_client = boto3.client('sts')
sts_response = sts_client.assume_role(RoleArn=target_role_arn, RoleSessionName='assume-role')
credentials=sts_response['Credentials']

s3_resource=boto3.resource(
    's3',
    aws_access_key_id=credentials['AccessKeyId'],
    aws_secret_access_key=credentials['SecretAccessKey'],
    aws_session_token=credentials['SessionToken'],
)

s3_resource.Bucket('bucket_kjshadjas_3432_dsadas').download_file('test_path/some_demo_file.gz', '/tmp/file.gz')

print(glob.glob("/tmp/*"))
```

The variable "target_role_arn" is the AWS ARN of the role to be assume, in this case we are assumiong a role in the same account but this role can be a cross account to access resources in a different account.

Once the role is assumed by the Glue job using the API call "assume_role" the role gets temporary credentials (AWS programatic keys) linked to the new role assumed. We can use those credentials to download S3 files and in this case it will succeeded because the role assumed has enough permissions to perform the task.