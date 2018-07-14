# Version 1 - the monolith

This is the first iteration on automating the resource provisioning process. This approach contains both "web" and "build pipeline" resources in the same template. 

## How it works

1. We need SSL certificate to be created/imported in N.Virginia region.
2. The infrastructure template is provided with the SSL certificate ARN, domain and sub domain names. All the resources are provisioned in one operation.
3. Some other steps are required in order to set up CodeCommit repository.
4. Test the process by pushing a change and expecting the website up and running.

## Prerequisites

AWS CLI, check it's installed
```shell
$ aws --version
aws-cli/1.15.59 Python/2.7.13 Linux/4.9.0-4-amd64 botocore/1.10.58
```
A default region in your profile i.e default
```shell
$ aws configure get region
eu-west-1
```
Or if using a named profile i.e serverless-web
```shell
$ aws configure get serverless-web.region
eu-west-1
```
See [AWS docs](https://docs.aws.amazon.com/cli/latest/userguide/cli-chap-getting-started.html) on how to configure CLI.

jq to parse JSON files, can be installed by running i.e ```sudo apt-get install jq```
```shell
$ jq --version
jq-1.5-1-a5b5cbe
```

Set current directory to ```templates/v1```

## SSL certificate in N.Virginia region

AWS Certificate Manager (ACM) - when we create a new certificate, it requires to validate domain ownership before issuing it. There are two ways to validate ownership:

### Use Email to Validate Domain Ownership

When using this option, ACM will send an email to the three registered contact addresses in WHOIS (Domain registrant, Technical contact, Administrative contact) then will wait for up to 72h for confirmation or it will time out. 

This approach requires manual intervention which is not great for automation although there might be scenarios where this is applicable. See [official AWS documentation](https://docs.aws.amazon.com/acm/latest/userguide/gs-acm-validate-email.html).

#### Shell commands

Set the variables for domain name and CloudFormation stack name
```shell
$ DOMAIN_NAME=abelperez.info
$ SSL_STACK_NAME=abelperez-info-ssl
```
Deploy ```ssl-certificate``` CloudFormation template in N.Virginia (us-east-1) region 

```shell
$ aws cloudformation deploy --stack-name $SSL_STACK_NAME \
--template-file ssl-certificate.yaml \
--parameter-overrides DomainName=$DOMAIN_NAME \
--region us-east-1
```
```
Waiting for changeset to be created..
Waiting for stack create/update to complete
Successfully created/updated stack - abelperez-info-ssl
```

At this point you must have received the emails from ACM and validated domain ownership. Next, get the SSL certificate ARN to be used in further steps.

```shell
$ SSL_CERT_ARN=`aws cloudformation describe-stacks --stack-name $SSL_STACK_NAME \
--query Stacks[0].Outputs \
--region us-east-1 \
| jq -r '.[] | select(.OutputKey == "SSLCertificateArn").OutputValue'`
```

Verify the certificate ARN 

```shell
$ echo $SSL_CERT_ARN
arn:aws:acm:us-east-1:923123123123:certificate/69fbad8c-xxxx-yyyy-zzzz-eb59a753c09c
```


### Use DNS to Validate Domain Ownership

When using this option, ACM will need to know that you have control over the DNS settings on the domain, it will provide a pair name/value to be created as a CNAME record which it will use to validate and to renew if you wish so. 

This approach is more suitable for automation since it doesn't require manual intervention. 
However, as of this writing, [it's not supported yet by CloudFormation](https://forums.aws.amazon.com/thread.jspa?messageID=821952) and therefore it will need to be done by using AWS CLI. Follow up the [official announcement and comments](https://aws.amazon.com/blogs/security/easier-certificate-validation-using-dns-with-aws-certificate-manager/). See [official AWS documentation](https://docs.aws.amazon.com/acm/latest/userguide/gs-acm-validate-dns.html).