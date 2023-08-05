# Automating AWS Security Hub Alerts with AWS Control Tower lifecycle events

*By Jason Cornick | on 01 NOV 2021 |by Jason Cornick | on 01 NOV 2021 | in [Amazon CloudWatch](https://aws.amazon.com/blogs/mt/category/management-tools/amazon-cloudwatch/), [AWS CloudFormation](https://aws.amazon.com/blogs/mt/category/management-tools/aws-cloudformation/), [AWS CloudTrail](https://aws.amazon.com/blogs/mt/category/management-tools/aws-cloudtrail/), [AWS Config](https://aws.amazon.com/blogs/mt/category/management-tools/aws-config/), [AWS Control Tower](https://aws.amazon.com/blogs/mt/category/management-tools/aws-control-tower/), [AWS Organizations](https://aws.amazon.com/blogs/mt/category/security-identity-compliance/aws-organizations/), [Intermediate(200)](https://aws.amazon.com/blogs/mt/category/learning-levels/intermediate-200/) , [Management Tools](https://aws.amazon.com/blogs/mt/category/management-tools/), [Technical How-To](https://aws.amazon.com/blogs/mt/category/post-types/technical-how-to/) | [Permalink](https://aws.amazon.com/blogs/mt/automating-aws-security-hub-alerts-with-aws-control-tower-lifecycle-events/) |  [Share](https://aws.amazon.com/blogs/mt/automating-aws-security-hub-alerts-with-aws-control-tower-lifecycle-events/#)*



**Important Update**: *As of 23 Nov 2020 the Security Hub service was updated to support direct integration with AWS Organizations. Lifecycle events are no longer the recommended way to enable Security Hub. Please utilize Security Hub’s native integration with AWS Organizations. You can also refer to [this blog](https://aws.amazon.com/blogs/mt/automating-amazon-guardduty-deployment-in-aws-control-tower/), which walks through how to enable GuardDuty in Control Tower using Delegated Administrator, which mirrors how you would setup Security Hub in Control Tower using Delegated Administrator.*

---

[AWS Control Tower](https://aws.amazon.com/controltower/) is an AWS managed service that automates the creation of a well-architected multi-account AWS environment. Control Tower simplifies new account provisioning for your [AWS Organization](https://aws.amazon.com/organizations/). Control Tower also centralizes logging from [AWS CloudTrail](http://aws.amazon.com/cloudtrail) and [AWS Config](http://aws.amazon.com/config), and provides preventative and detective [guardrails](https://aws.amazon.com/blogs/mt/how-to-detect-and-mitigate-guardrail-violation-with-aws-control-tower/).

[AWS Security Hub](https://aws.amazon.com/security-hub) can be used to provide a comprehensive view of high-priority security alerts and compliance status across AWS accounts. Bringing these two solutions together allows customers to govern and launch cloud capabilities with an aggregated security view within their multi-account framework.

This post walks through the process of automating Security Hub enablement and configuration in a Control Tower multi-account managed environment using [Control Tower lifecycle Events](https://aws.amazon.com/blogs/mt/using-lifecycle-events-to-track-aws-control-tower-actions-and-trigger-automated-workflows/). This solution also enables and configures Security Hub on any new account provisioned or updated from the [Account Factory](https://docs.aws.amazon.com/controltower/latest/userguide/account-factory.html) and runs a scheduled task to ensure that all accounts stay enabled. Automating this feature helps reduce complexity and risk, enhancing your security posture while saving time and reducing operational burden.

## Solution overview 

Security Hub helps you monitor critical settings to ensure that your AWS accounts remain secure, allowing you to detect and react quickly to any security event in your environment.

Enabling Security Hub in an AWS account requires just a few selections in the AWS Management Console. Once enabled, Security Hub begins aggregating and prioritizing findings and conducting compliance checks. In a multi-account environment, Security Hub findings from all member accounts can be aggregated to a Security Hub management account, which allows effective monitoring of critical security events across the organization. Aggregated findings across the organization are also available in the [Amazon CloudWatch](http://aws.amazon.com/cloudwatch) Events of the Security Hub management account, enabling you to automate your AWS services and respond automatically to events.

When working in a multi-account environment, the Security Hub management account sends an invitation to member accounts. These member accounts must then accept the invitation, which grants permissions to the management account to view the findings of each of the member accounts. In this solution, we automate this process by triggering [AWS Lambda functions](http://aws.amazon.com/lambda)  when Control Tower provisions new accounts. This guide assumes that you have Control Tower already set up. If you have not set up Control Tower yet, follow the steps on [Getting Started with AWS Control Tower guide](https://docs.aws.amazon.com/controltower/latest/userguide/getting-started-with-control-tower.html).

The architecture for this solution is mapped in the following diagram.

![image](https://d2908q01vomqb2.cloudfront.net/972a67c48192728a34979d9a35164c1295401b71/2020/03/20/1-SolutionArchitecture.png)
*Fig 1 Solution Architecture*

Here is an overview of how the solution works:  
* The *SecurityHubEnablerLambda* enables Security Hub in the Security Hub management Account.  
* The Lambda function loops through all supported Regions in the member accounts, assuming a role, and enabling Security Hub and the [Center for internet Security (CIS) AWS Foundations Benchmark](https://d0.awsstatic.com/whitepapers/compliance/AWS_CIS_Foundations_Benchmark.pdf). There are [other compliance standards](https://docs.aws.amazon.com/securityhub/latest/userguide/securityhub-standards.html) that can be enabled, but the [CIS Benchmark](https://d0.awsstatic.com/whitepapers/compliance/AWS_CIS_Foundations_Benchmark.pdf) has been selected as a useful starting point for most organizations.  
* Member accounts are registered in the Security Hub management account and an invitation is sent to the member accounts.  
* The invitation is accepted in the member accounts.  
* When new accounts are successfully vended from Account Factory, a Control Tower Lifecycle Event triggers the Lambda function to loop through the process again.

## Prerequisites
This solution assumes that you have access to the Control Tower management account. We deploy to the Control Tower management account and leverage the AWSControlTowerExecution role from Control Tower.

Although you deploy this solution from your Control Tower management account, you must choose an AWS account as your Security Hub management. By default, Control Tower creates a security Audit account for cross-account auditing and centralized security operations within the Control Tower Organization. We use the Audit account for our Security Hub management account, but if you have another account you think is more appropriate, then use it instead.

Before launching this solution, you will must gather the following information:

• The ID of the Organization (in a format such as o-xxxxxxxxxx) and the email address of the owner of its management account, which can be found on the [Settings tab of the Organizations console](https://docs.aws.amazon.com/organizations/latest/userguide/orgs_manage_org_details.html#orgs_view_org).
• The Account ID of the account you would like to be your Security Hub management account. From the AWS Organizations console, select Accounts. The Account ID is in a 12-digit numeric format.
• An [Amazon S3](http://aws.amazon.com/s3) bucket to host the Lambda package. Identify or [create a bucket](https://docs.aws.amazon.com/AmazonS3/latest/user-guide/create-bucket.html), and take note of that bucket’s name. The Amazon S3 bucket must be in the same Region in which you plan launch AWS CloudFormation, and should be a [Region supported by Control Tower](https://aws.amazon.com/controltower/faqs/#Availability).
• Select this [AWS CloudFormation template](https://raw.githubusercontent.com/aws-samples/aws-control-tower-securityhub-enabler/main/aws-control-tower-securityhub-enabler.template). Save it to your computer so you can upload it later.

## Packaging the Lambda code
In this solution, we provide you with sample Lambda code. You can also clone the GitHub [repo](https://github.com/aws-samples/aws-control-tower-securityhub-enabler) to tailor the code to your needs and contribute. The Lambda function is written in Python, but is too large to be included in-line in the CloudFormation template. To deploy it, you must [package it](https://docs.aws.amazon.com/lambda/latest/dg/python-package.html).

1. Start by ensuring you have installed a supported version of Python (3.5+). If you don’t already have it, the latest version can be downloaded [here](https://www.python.org/downloads/windows/)  
2. Next, download the code [here](https://github.com/aws-samples/aws-control-tower-securityhub-enabler/releases/latest) or clone the repo from [GitHub](https://github.com/aws-samples/aws-control-tower-securityhub-enabler). Decompress if you downloaded the ZIP version.
3. Open a command prompt.
4. Navigate to the folder into which you extracted the zip file or cloned the repo and run <blockquote style="background-color: #f2f2f2; color: #b80000; padding: 0.2em 1em; margin : 0.6em 1em; display:inline">cd src</blockquote> to change to the subfolder. There is a package script in the folder that reads the dependencies from requirements.txt, downloads them, and then creates a zipped package for Lambda.  

Bash
<blockquote style="background-color: #111111; color: #e0a82f; border-left: 10px solid #000000; padding: 0.5em 1em;">
./package.sh
</blockquote>  
  

###

###  

PowerShell
<blockquote style="background-color: #111111; color: #e0a82f; border-left: 10px solid #000000; padding: 0.5em 1em;">
./package.ps1
</blockquote>
  

## Uploading the Lambda code
Now, upload the ZIP package you built into the Amazon S3 bucket that you created earlier.
  

1. Log in to your Control Tower management account as a user or role with administrative privileges.  

2. Upload the securityhub-enabler.zip you created into your Amazon S3 bucket. Make sure you upload it to the root of the bucket.

## Launching the AWS CloudFormation stack
Next, launch the AWS CloudFormation stack.

1. Go to AWS CloudFormation in the AWS Management Console.
2. Confirm that your console session is in the same Region as the Amazon S3 bucket in which you stored the code.
3. Choose **Create Stack** and select **With new resources (standard).**
4. On the **Create Stack** page, under **Specify template**, select the **Upload a template file** template source.
5. Select **Choose file** and find the template you downloaded in the prerequisites steps.
6. Choose **Next** to continue.
7. On the **Specify Stack Details** page, give your stack a name such as “MySecurityHubStack.”
8. Under **Parameters**, review the default parameters and enter the required <font color="red">*OrganizationID*</font>, <font color="red">*S3SourceBucket*</font>, and <font color="red">*SecurityAccountID*</font> gathered earlier.  

There are several parameters for the AWS CloudFormation stack:
- **ComplianceFrequency**: Frequency in days to re-run Lambda to check Security Hub compliance across all targeted accounts.
- **OUFilter**: OU Scope for Security Hub deployment; choose either all OUs or Control Tower managed OUs.
- **OrganizationID**: AWS Organizations ID for Control Tower. This is used to restrict permissions to least privilege.
- **RegionFilter**: Region scope for Security Hub deployment; choose either All AWS Regions or Control Tower managed Regions.
- **RoleToAssume**: [IAM](http://aws.amazon.com/iam) role to be assumed in child accounts to enable Security Hub; the default is <font color="red">*AWSControlTowerExecution*</font>. If you’re deploying to a non-Control Tower managed account, make sure that this role exists in all accounts.
- **S3SourceBucket**: Amazon S3 bucket containing the securityhub_enabler.zip file you uploaded earlier.
- **SecurityAccountId**: The AWS account ID of your Security Hub management account.  


1. On the **Configure stack** options page you can choose to add tags, choose additional options, or just choose Next.
2. On the Review page, validate your parameters and acknowledge that IAM resources will be created. Finally, select **Create stack**.
Once you select **Create stack**, you can follow the process and view the status of the deployment via the **Events** tab of the CloudFormation stack. When it finishes deploying, move on to the next step.

## Validating in the Security Hub console  
CloudFormation automatically triggers the Lambda function to invite all existing AWS Accounts in Control Tower to join the Security Hub. The **OUFilter** parameter in the CloudFormation stack determines the scope of deployment.

After the stack has completed deploying from the Control Tower management account, sign in to the AWS Management Console of the Security Hub management account and open the AWS Security Hub console.

1. From the Security Hub Summary page, you see the consolidated insights and compliance standards within minutes, as shown in the following screenshot.  
![A snapshot of the Security Hub Summary page showing the compliance for a Control Tower Environment](https://d2908q01vomqb2.cloudfront.net/972a67c48192728a34979d9a35164c1295401b71/2020/03/24/SH_Summary.png)
*Fig 2 Security Hub Summary*  




2. Under **Settings**, you can also view the member accounts now sharing their findings with you, as shown in the following screenshot.  
![The Settings page of the AWS Security Hub console showing the member account associations.](https://d2908q01vomqb2.cloudfront.net/972a67c48192728a34979d9a35164c1295401b71/2020/03/20/3-Settings.png)
*Fig 2 Security Hub Summary* 


From this point forward, any newly vended accounts from Account Factory will automatically have Security Hub enabled and share their findings with this Audit account.

## Cleaning up
To avoid incurring future charges, you can disable Security Hub by deleting the resources created. You can delete the provisioned CloudFormation stack via the [AWS CloudFormation](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/cfn-console-delete-stack.html) console or the command line. Deleting the CloudFormation stack triggers the Lambda function to disable Security Hub in all the accounts, as well as cleans up the resources deployed by the stack.

## Conclusion
In this post, we demonstrated how to leverage [Control Tower Life Cycle Events](https://docs.aws.amazon.com/controltower/latest/userguide/lifecycle-events.html) to enable and configure Security Hub in a Control Tower environment. Control Tower already provides customers with the ability to centrally manage compliance through guardrails. With this solution, customers can now also leverage Security Hub to get a comprehensive view of their high-priority security alerts and compliance status across all the AWS accounts in their landing zone environment. Furthermore, you can leverage Security Hub findings in additional ways to enhance your security posture, including:

- Invoking a Lambda function
- Invoking an [Amazon EC2](http://aws.amazon.com/ec2) run command
- Relaying the event to [Amazon Kinesis Data Streams](https://aws.amazon.com/kinesis)
- Notifying an [Amazon SNS topic](https://aws.amazon.com/sns) or an [AWS SQS queue](https://aws.amazon.com/sqs)
- Sending a finding to a third-party ticketing, chat, SIEM, or incident response and management tool  

## Further Information
- [Security Hub users guide](https://docs.aws.amazon.com/securityhub/latest/userguide/what-is-securityhub.html)
- [Multi-account Security Hub](https://docs.aws.amazon.com/securityhub/latest/userguide/securityhub-accounts.html)
- [Control Tower governance](https://docs.aws.amazon.com/controltower/latest/userguide/what-is-control-tower.html)
- [Service Catalog governance](https://docs.aws.amazon.com/servicecatalog/latest/adminguide/introduction.html)
- [Multi-account framework](https://www.youtube.com/watch?v=zVJnenaD3U8)

## About the Authors  



![Jason Cornick](https://d2908q01vomqb2.cloudfront.net/972a67c48192728a34979d9a35164c1295401b71/2020/03/20/4-JasonCornick-150x150.png)  
Jason Cornick is a Senior Cloud Infrastructure Architect for AWS Professional Services. He has been working with AWS technology for more than 4 years. Jason works with Enterprise customers to accelerate cloud adoption through building Landing Zones and transforming IT organizations to adopt cloud operating practices and agile operations. When not working he enjoys kayaking and reading science fiction.
  
## 

![Welly Siauw](https://d2908q01vomqb2.cloudfront.net/972a67c48192728a34979d9a35164c1295401b71/2020/03/20/5-WellySiauw-150x150.png)  
Welly Siauw is a Senior Technical Account Manager at AWS Enterprise Support. Welly enjoys working with AWS customers in solving architectural, operational, and cost optimization challenges.

#
![Jason Moldan](https://d2908q01vomqb2.cloudfront.net/972a67c48192728a34979d9a35164c1295401b71/2020/03/20/4-JasonCornick-150x150.png)  
Jason Moldan is a Solutions Architect for AWS World Wide Public Sector where he works with government customers to implement cloud solutions and solve technical problems.

**TAGS**: [AWS Control Tower](https://aws.amazon.com/blogs/mt/tag/aws-control-tower/), [AWS Security Hub](https://aws.amazon.com/blogs/mt/tag/aws-security-hub/)

## Credits:
[Automating AWS Security Hub Alerts with AWS Control Tower lifecycle events](https://aws.amazon.com/blogs/mt/automating-aws-security-hub-alerts-with-aws-control-tower-lifecycle-events/)
