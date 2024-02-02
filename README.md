# CDKLP - AWS CDK Least Privilege Policy Generator
`cdklp` is a CLI tool designed to generate least privilege IAM policies for CloudFormation templates created by the AWS Cloud Development Kit (CDK). This tool is based on the **aws-leastprivilege open-source project,** which aims to automate the creation of minimally privileged IAM policies for various AWS services.

By analyzing the resources defined in a CDK-generated CloudFormation template, cdklp efficiently constructs IAM policies that grant only the permissions necessary for the resources to operate correctly. This approach helps in maintaining a secure AWS environment by adhering to the principle of least privilege.

For more information about the original project, [visit aws-leastprivilege on GitHub.](https://github.com/iann0036/aws-leastprivilege)

## Problem Statement

When deploying AWS resources using the AWS Cloud Development Kit (CDK), it's crucial to adhere to the principle of least privilege in IAM roles. However, determining the exact set of minimum permissions required for a specific CDK deployment can be challenging. Often, the IAM roles assume broader permissions than necessary, posing a potential security risk.

## Solution

This script aims to generate a minimal IAM policy based on the resources defined in a CDK-generated CloudFormation template. By analyzing the resource types and names within the template, the script dynamically constructs an IAM policy that grants only the necessary permissions for the creation of those resources.

## How It Works

The script operates in two main stages:

1. **Extracting Resource Information**: It reads a CloudFormation template (in YAML format) and extracts the types and identifiers of the resources defined in the template.
2. **Generating IAM Policy**: Based on the extracted information, the script then generates an IAM policy. This policy includes permissions tailored to the specific needs of the resources identified in the first stage.

## Install cdklp CLI
1. Clone this repository.
   `git clone https://github.com/jimini55/cdk-leastprivilege.git` 
2. (Optional) Create virtual environment.
   `python -m venv venv`
3. Install requirements.
   `pip install -r requirements.txt`
4. Install `cdklp` CLI locally by running the command.
   `pip install .`


## cdklp CLI usage
To use the script, follow these steps:

1. Ensure you have a synthesized CloudFormation template from your CDK application (usually generated using `cdk synth >> template.yaml`).
2. Run the script with the path to the CloudFormation template. The script will output a JSON formatted IAM policy. Example:
   `cdklp -i /path/to/your_template.yaml --profile default --region us-west-1 --consolidate-policy > result.json`
3. Review and adjust the generated policy as necessary before applying it to your IAM roles.

### Options

The following command line arguments are available:

#### -i, --input-filename <filename>

The filename of a local CloudFormation template file to analyze. 


#### --consolidate-policy

When specified, the `Sid` fields will be removed and statements sharing the same attributes except `Action` will be combined.

#### --region <name>

Overrides the region to specify in policy outputs and when retrieving deployed templates. By default, the region will be retrieved using the [default precedence](https://boto3.amazonaws.com/v1/documentation/api/latest/guide/configuration.html#configuring-credentials) for Boto3.

#### --profile <name>

When specified, the specified named profile credentials will be used for all data gathering AWS actions. The `AWS_PROFILE` environmental variable would also be respected if this property is not set.



## Policy Generation Logic

Policies will be created with data following the below preference:
1. Per-type mappings created by incrementally increasing required permissions
2. Permissions retrieved from the [CloudFormation Registry](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/registry.html)
3. No data available (a warning will be shown for missed types)

### For supported per-type mapping resources

The generated policy will be as specific as possible when specifying actions, resources and conditions. Wildcard actions are never used and all conditions that are available will be populated unless:

* The condition would take no effect or there is not enough information to specify the condition, or
* The condition is a [global condition](https://docs.aws.amazon.com/IAM/latest/UserGuide/reference_policies_condition-keys.html), or
* The condition applies to an update statement and would prevent the field from being freely changed, or
* The condition relates to the tag keys/values

Resources may be fully or partially wildcarded however will be as specific as possible.

Update statements are disabled by default. If enabled with the `--include-update-actions` option, only properties that have a value specified in the template will have an associated update statement that allows that value to be changed. Permissions required to add new properties may not have the permissions included in the policy.

### For permissions retrieved from the CloudFormation Registry

The generated policy will only include the actions specified in the resource type specification provided by the registry. All resources will be wildcarded and no conditions will apply.

## Supported Resource Types

The following resource types are supported with a per-type mapping:

* AWS::CloudWatch::Alarm
* AWS::EC2::Instance
* AWS::EC2::SecurityGroup
* AWS::EC2::Subnet
* AWS::EC2::VPC
* AWS::IAM::Role
* AWS::Lambda::Function
* AWS::Lambda::Version
* AWS::Route53::HostedZone
* AWS::S3::Bucket
* AWS::SNS::Topic
* AWS::SQS::Queue

## Example
- `synth.yaml` is a cloudformation template generated by `cdk synth`.
- The example policy for this can be found in `result.json`. It has the exact resource name in `Resource` and the minimal actions set as `Allow`.
  
```
...
    {
        "Effect": "Allow",
        "Action": [
            "sqs:CreateQueue",
            "sqs:DeleteQueue",
            "sqs:GetQueueAttributes"
        ],
        "Resource": "arn:aws:sqs:us-west-1:012345689910:sfcc-destination-destination-event-queue"
    },
...
```

## Cleanup
`pip uninstsall cdklp`

## Note

This script is intended as a starting point for generating IAM policies. It's important to review the generated policies to ensure they align with your specific AWS environment and security requirements.

## Q&A

### Will cdklp address all IAM security concerns for developers?

While cdklp significantly aids in deploying CDK with the least privilege, it's important to understand the role of [permission boundaries.](https://docs.aws.amazon.com/IAM/latest/UserGuide/access_policies_boundaries.html) [Setting a permission boundary for the CDK exec role is a best practice,](https://aws.amazon.com/blogs/devops/secure-cdk-deployments-with-iam-permission-boundaries/), however, cdklp is invaluable in identifying the exact permissions required for developers, which is crucial for navigating internal security approval processes, especially in environments with strict IAM policies.

### How does cdklp differ from the original open-source project?

The key differences stem from handling CDK metadata, which isn't processed by the original open-source project. cdklp includes logic to remove CDK metadata. Additionally, based on tests with some CDK packages, I've incorporated default IAM permissions like iam:GetRole and ssm:GetParameters into the policy template. 

### Does it support all CDK projects?

Currently, cdklp doesn't support all AWS resources, meaning it can't generate the least privilege IAM policy for every scenario. 