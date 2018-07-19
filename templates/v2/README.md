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
DOMAIN_NAME=abelperez.info
SUB_DOMAIN_NAME=www
BASE_STACK_NAME=abelperez-info-base
INCLUDE_REDIRECT=true
```

Deploy base stack

```shell
aws cloudformation deploy --stack-name $BASE_STACK_NAME \
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
SSL_CERT_ARN=arn:aws:acm:us-east-1:923123123123:certificate/69fbad8c-xxxx-yyyy-zzzz-eb59a753c09c
DOMAIN_NAME=abelperez.info
SUB_DOMAIN_NAME=test
BASE_STACK_NAME=abelperez-info-base
INCLUDE_REDIRECT=true
```

Deploy base infra stack.

```shell
aws cloudformation deploy --stack-name $BASE_STACK_NAME \
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


## Deploy the build template


## Resources configuration

Get the outputs from web and build templates

commands ...



(as per v1)
### Upload SSH key to CodeCommit user
### Configure SSH host names
### Clone the repository
### Put some content in the repository
## Start using the repository
## Testing end to end process
