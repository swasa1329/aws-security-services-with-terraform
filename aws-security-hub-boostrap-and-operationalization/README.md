## AWS Security Hub - Bootstrap and Operationalization
![Architecture](https://github.com/aws-samples/aws-security-services-with-terraform/blob/master/aws-security-hub-boostrap-and-operationalization/Architecture.jpg)

* ***NOTE*** These config files have been tested with Terraform v.0.11.14 and the AWS Provider v2.45

The Terraform configuration files within this folder are designed to help customers bootstrap and operationalize Security Hub by enabling downstream services (Config, GuardDuty, Inspector), creating resources that are compliant with AWS CIS Foundations Benchmark controls and extending Security Hub via CloudWatch and Kinesis by consuming all findings into Elasticsearch Service and using Kibana as a Security Information & Event Management (SIEM) tool. All Terraform config files are created seperately to offer modularity (though all variables are within `variables.tf`) if you already have certain resources deployed, or you wanted to craft your own Terraform config files. You can reuse your same `provider.tf` file from the WAF Blog for this solution as well to retain your state in your remote backend.

## Required IAM Permissions
You will need to modify the CodeBuild role created for the WAF Blog to allow it to create all of these resources, which include the below. You need to give Terraform full create/update/read/delete permissions to be able to apply, update, describe and delete your State.
- CloudWatch Events
- CloudWatch Logs
- CloudTrail
- Config
- Cognito
- EC2
- Elasticsearch Service
- GuardDuty
- IAM
- Inspector
- Kinesis Data Streams
- Kinesis Data Firehose
- KMS
- Lambda
- S3
- SNS
- VPC

## Terraform Configuration File Inventory
### variables.tf
A majority of the variables in this config file are already filled out with `default` values. Comments are added to denote which variables belong to which config file.

### config.tf
This config file will create all resources needed to create and enable an AWS Config recorder, if you already have one the state applied with this config file will fail.

**NOTE** You may still need to go through the Config console and hit save for Config to be enabled.

### security_services.tf
This config file will enable a GuardDuty, Security Hub and create an Inspector template and target group that will target all EC2 instances with all assessment templates. These assessment templates are provided via a Terraform variable list, ensure you specify the correct one. Only us-east-1, us-west-1 and ap-southeast-2 templates are created, you'll need to create your own for other regions, refer to [Amazon Inspector ARNS for Rules Packages](https://docs.aws.amazon.com/inspector/latest/userguide/inspector_rules-arns.html) for more information

**NOTE** Depending on the timing of your resources being created by Terraform, Config and Security Hub (if using both `config.tf` and `security_services.tf`), the Security Hub Service-Linked Rules (SLRs) for Config used for the CIS AWS Foundations Benchmark controls may not created. To remediate this, disable the compliance standard and re-enable it within Security Hub. Don't forget to turn off controls that are not relevant to you.

### cis_baseline_infrastructure.tf
This config file will create a multi-region CloudTrail trail along with a S3 bucket, CloudWatch Logs group, IAM Password Policy and Server Access Logging S3 Bucket that are all in compliance with CIS 1.x and 2.x controls. Additionally a VPC with private and public subnets, VPC flow logging and an empty default SG (compliant with CIS 4.3) and all IAM roles and policies needed will be created. This will help you be in compliance with CIS and can serve as a baseline for a new environment being created. The amount of AZs that will be created is dependnet on a variable named `Network_Resource_Count`, ensure you do not specify a value higher than the amount of AZs in your Region (i.e. 6 in us-east-1, 3 in us-east-2, etc.)

### cis_3-x.tf
This config file will create a SNS topic and all needed metric filters and alarms needed to be in compliance with CIS 3.x controls. This depends on the CloudWatch group that was created in `cis_baseline_infrastructure.tf`, if you decide to not use that file you will need to specify your own CloudWatch Log Group that CloudTrail publishes to.

**NOTE** Due to the way Terraform provisions resources, Email SNS subscriptions are not allowed to be created. **You must manually subscribe and accept an email subscription to the SNS topic for the Security Hub 3.x controls to be compliant.**

### security_hub_siem.tf
This config file will create an ElasticSearch Service domain, Cognito resources (for use for signing into Kibana) and a delivery pipeline that includes a CloudWatch Event Rule, Kinesis Data Stream and Kinesis Data Firehose Delivery Stream to send all findings from Security Hub to Elastic for exploration and analysis in Kibana. All necessary IAM roles and event patterns are created in this configuration file, you will need to modify the Firehose resource if you would rather send logs to S3 or to Splunk.

### elasticsearch_siem_extension.tf
This config file creates infrastructure that will send VPC Flow Logs and CloudTrail logs from CloudWatch to the Elasticsearch Service domain created by `security_hub_siem.tf`. This pipeline uses CloudWatch Log subscriptions, Lambda functions, Kinesis Data Firehose Delivery Streams and IAM roles / policies to accomplish the task.

## Troubleshooting
#### Elasticsearch Service timeout
Depending on the amount of nodes, Masters and the instance type Terraform may time out due to how long the Domain takes to deploy. Wait for it to be `Active` in the AWS Management Console and then re-apply state after it has completed to avoid race conditions or orphaned resources.

**NOTE** The default settings of 2 instances, 3 masters and c4.large instances should take around 15 minutes to create and around 7 minutes to destroy

#### Terraform v.0.12.x Support
These Config files are written to v0.11.14, as was supported by the original WAF Terraform CI/CD blog, these will not work for v0.12.x without modifications

#### Error creating <resource>: ValidationException: 
Terraform has been observed throwing validation errors when creating a large amount of resources (this solution is over 100 resources), if you are running in CI/CD release the change again to deploy the missing state. If you are running from your local computer call `terraform apply` again.

#### aws_inspector_assessment_template.Inspector_Assessment_Template: NoSuchEntityException
Ensure you are using the Assessment Template variable that matches the region you are deploying to. Currently, only `us-east-1` (North Virginia), `us-west-1` (North California), `ap-southeast-2` (Sydney, Australia) and `eu-west-2` (London, UK) are supported.

If you add other regional support, please open a PR to have them added to mainline. If you need help creating them please open an Issue.

## License

This library is licensed under the MIT-0 License. See the LICENSE file.