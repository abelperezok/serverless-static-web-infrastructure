# Version 2 - Web and Build Pipeline in separate templates and Lambda@Edge for redirection rules

This is the second iteration based on the lessons learned in the previous iteration. This approach segregates "web infrastructure" from "build infrastructure" in different templates. Also uses Lambda@Edge to run a simple logic execute the redirection rule, as opposed to duplicating S3 buckets and CloudFront distributions.

## How it works

* We need SSL certificate to be created/imported in N.Virginia region. It can be created conditionally by base template.
* The base template which provisions the Lambda@Edge function needs to be created in N.Virginia region.
* The web template is provided with the SSL certificate ARN, domain and sub domain names.
* The build template is provided only with the bucket name, it provisions all the resources based on that.
* Some other steps are required in order to set up CodeCommit repository.
* Test the process by pushing a change and expecting the website up and running.

## Prerequisites
(same as v1) - provide some link...


Set current directory to ```templates/v2```

## SSL certificate in N.Virginia region

In this version, the base infrastructure is required if you want to validate the SSL certificate via Email or if you want to include the canonical redirection rule (non-www to www). If you go for DNS validation and do not require canonical redirection rule, you can skip using this stack completely.

### Validating certificate using Email

if validating certificate using Email, it can be created by base template, in this case, do not set the SSLCertificateArn parameter

Set the variables to store the initial values:

```shell
$ DOMAIN_NAME=abelperez.info
$ SUB_DOMAIN_NAME=www
$ BASE_STACK_NAME=abelperez-info-base
$ INCLUDE_REDIRECT=true
```

Deploy base stack

```shell
$ aws cloudformation deploy --stack-name $BASE_STACK_NAME \
--template-file base-infra.yaml \
--capabilities CAPABILITY_IAM \
--parameter-overrides \
DomainName=$DOMAIN_NAME \
SubDomainName=$SUB_DOMAIN_NAME \
IncludeRedirectToSubDomain=$INCLUDE_REDIRECT \
--region us-east-1

Waiting for changeset to be created..
Waiting for stack create/update to complete
Successfully created/updated stack - abelperez-info-base
```

### Validating certificate using DNS

if validating certificate using DNS, it has to created via cli and provide the certificate ARN to base template. (see v1 -- provide some link)

Once we have the certificate already validated, deploy the base infra stack. First, set the variables to store the initial values.

```shell
$ SSL_CERT_ARN=arn:aws:acm:us-east-1:923123123123:certificate/69fbad8c-xxxx-yyyy-zzzz-eb59a753c09c
$ DOMAIN_NAME=abelperez.info
$ SUB_DOMAIN_NAME=test
$ BASE_STACK_NAME=abelperez-info-base
$ INCLUDE_REDIRECT=true
```

Deploy base infra stack.

```shell
$ aws cloudformation deploy --stack-name $BASE_STACK_NAME \
--template-file base-infra.yaml \
--capabilities CAPABILITY_IAM \
--parameter-overrides \
DomainName=$DOMAIN_NAME \
SubDomainName=$SUB_DOMAIN_NAME \
SSLCertificateArn=$SSL_CERT_ARN \
IncludeRedirectToSubDomain=$INCLUDE_REDIRECT \
--region us-east-1

Waiting for changeset to be created..
Waiting for stack create/update to complete
Successfully created/updated stack - abelperez-info-base
```

## Deploy the web template

Before deploying web stack, it's required to capture the outputs from the base infra stack, as some parameters will depend on that. First the the outputs section from base stack.

```shell
$ BASE_STACK_OUTPUT=`aws cloudformation describe-stacks \
--stack-name $BASE_STACK_NAME \
--query Stacks[0].Outputs \
--region us-east-1`
```
If you are not using canonical redirection, LambdaEdgeRedirectFunction parameter is not required.

Extract the Lambda@Edge function ARN including version, this is important for CloudFront distribution to identify the published version of the Lambda function to connect to the Viewer Request event,

```shell
$ LAMBDA_ARN_VERSION=`echo $BASE_STACK_OUTPUT \
| jq -r '.[] | select(.OutputKey == "LambdaEdgeRedirectFunctionIncludingVersion").OutputValue'`
```

Assuming the values for the variables from the previous steps. Add this new variable.

```shell
$ WEB_STACK_NAME=abelperez-info-web
```

Deploy web stack.

```shell
$ aws cloudformation deploy --stack-name $WEB_STACK_NAME \
--template-file web-infra.yaml \
--capabilities CAPABILITY_IAM \
--parameter-overrides \
DomainName=$DOMAIN_NAME \
SubDomainName=$SUB_DOMAIN_NAME \
SSLCertificateArn=$SSL_CERT_ARN \
IncludeRedirectToSubDomain=$INCLUDE_REDIRECT \
LambdaEdgeRedirectFunction=$LAMBDA_ARN_VERSION

Waiting for changeset to be created..
Waiting for stack create/update to complete
Successfully created/updated stack - abelperez-info-web
```

## Deploy the build template

Before deploying web stack, it's required to capture the outputs from the base infra stack, as some parameters will depend on that. First the the outputs section from base stack.

```shell
$ WEB_STACK_OUTPUT=`aws cloudformation describe-stacks \
--stack-name $WEB_STACK_NAME \
--query Stacks[0].Outputs`
```
Extract the S3 bucket name and ARN which will be used in further steps.

```shell
$ S3_BUCKET_NAME=`echo $WEB_STACK_OUTPUT \
| jq -r '.[] | select(.OutputKey == "S3StaticBucketName").OutputValue'`

$ S3_BUCKET_ARN=`echo $WEB_STACK_OUTPUT \
| jq -r '.[] | select(.OutputKey == "S3StaticBucketArn").OutputValue'`
```

Set the build stack name variable.

```shell
$ BUILD_STACK_NAME=abelperez-info-build
```

Deploy build infra stack, the only required parameter is the S3 bucket ARN where static HTML file will be copied as a result of the build pipeline. 

```shell
$ aws cloudformation deploy --stack-name $BUILD_STACK_NAME \
--template-file build-infra.yaml \
--capabilities CAPABILITY_IAM CAPABILITY_NAMED_IAM \
--parameter-overrides StaticSiteBucketArn=$S3_BUCKET_ARN

Waiting for changeset to be created..
Waiting for stack create/update to complete
Successfully created/updated stack - abelperez-info-build
```

## Resources configuration

Get the outputs from web and build templates. From Web infra stack $S3_BUCKET_NAME is already set with the required value. 

```shell
$ BUILD_STACK_OUTPUT=`aws cloudformation describe-stacks \
--stack-name $BUILD_STACK_NAME \
--query Stacks[0].Outputs`
```

Extract CodeCommit SSH clone url and CodeCommit user name.

```shell
$ CC_SSH_URL=`echo $BUILD_STACK_OUTPUT \
| jq -r '.[] | select(.OutputKey == "CodeCommitRepositoryCloneUrlSsh").OutputValue'`

$ CC_USER_NAME=`echo $BUILD_STACK_OUTPUT \
| jq -r '.[] | select(.OutputKey == "CodeCommitUserName").OutputValue'`
```

(the rest is as per v1)

### Upload SSH key to CodeCommit user
### Configure SSH host names
### Clone the repository
### Put some content in the repository
## Start using the repository
## Testing end to end process

## Cleaning up stacks

To get rid of all the created resources, the most common way is to delete the stacks via CloudFormation. However, since we've manually modified some resources, it would fail the deletion process for two reasons:

* The IAM user (created for CodeCommit) has an SSH key associated and therefore needs to be removed before removing the user. CloudFormation would say something like: **"Cannot delete entity, must remove referenced objects first. (Service: AmazonIdentityManagement; Status Code: 409; Error Code: DeleteConflict; Request ID: 14070f65-878f-11e8-bcdc-138f1db22f46)"**

* The S3 bucket where we store the static HTML files is not empty and therefore cannot be deleted with the stack. CloudFormation would say something like: **"The bucket you tried to delete is not empty (Service: Amazon S3; Status Code: 409; Error Code: BucketNotEmpty; Request ID: C6F5F29EA58BFC3D; S3 Extended Request ID: VjLcE22uYD0f2xkoUpui/KGvkFE2Lb83GWqNFbFv8UZIHhGQ8Lc1UUCeOT5KP616yXAEluLU8L8=)"**

The following steps should be executed to safely delete all resources:

Remove the SSH key associated with the IAM user.

```shell
$ aws iam delete-ssh-public-key \
--user-name $CC_USER_NAME \
--ssh-public-key-id $CC_SSH_KEY_ID
```

Empty the S3 bucket.

```shell
$ aws s3 rm s3://$S3_BUCKET_NAME --recursive
```

The "delete-stack" command only sends the command "Delete Stack" to CloudFormation, deletion process can take some time, to make sure it's completed, we use the wait command.

Delete the build stack via CloudFormation cli.

```shell
$ aws cloudformation delete-stack \
--stack-name $BUILD_STACK_NAME
```

Wait for the build stack to delete.

```shell
$ aws cloudformation wait stack-delete-complete \
--stack-name $BUILD_STACK_NAME
```

Delete the web stack via CloudFormation cli.

```shell
$ aws cloudformation delete-stack \
--stack-name $WEB_STACK_NAME
```

Wait for the web stack to delete.

```shell
$ aws cloudformation wait stack-delete-complete \
--stack-name $WEB_STACK_NAME
```

When build and web stacks are completely deleted, it's time to delete the base stack in N. Virginia region.

```shell
$ aws cloudformation delete-stack \
--stack-name $BASE_STACK_NAME \
--region us-east-1
```

Once more, wait for the stack to be completely deleted.

```shell
$ aws cloudformation wait stack-delete-complete \
--stack-name $BASE_STACK_NAME \
--region us-east-1
```