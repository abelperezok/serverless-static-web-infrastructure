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
and verify that it's issued

```shell
$ aws acm describe-certificate \
--certificate-arn $SSL_CERT_ARN \
--query Certificate.Status \
--region us-east-1
"ISSUED"
```


### Use DNS to Validate Domain Ownership

When using this option, ACM will need to know that you have control over the DNS settings on the domain, it will provide a pair name/value to be created as a CNAME record which it will use to validate and to renew if you wish so. 

This approach is more suitable for automation since it doesn't require manual intervention. 
However, as of this writing, [it's not supported yet by CloudFormation](https://forums.aws.amazon.com/thread.jspa?messageID=821952) and therefore it will need to be done by using AWS CLI. Follow up the [official announcement and comments](https://aws.amazon.com/blogs/security/easier-certificate-validation-using-dns-with-aws-certificate-manager/). See [official AWS documentation](https://docs.aws.amazon.com/acm/latest/userguide/gs-acm-validate-dns.html).

Set the variables for domain name 
```shell
$ DOMAIN_NAME=abelperez.info
```

Request the certificate to AWS ACM 
```shell
$ SSL_CERT_ARN=`aws acm request-certificate \
--domain-name $DOMAIN_NAME \
--subject-alternative-names *.$DOMAIN_NAME \
--validation-method DNS \
--query CertificateArn \
--region us-east-1 \
--output text`
```

At this point we have the certificate but it's not validated yet. ACM provides values for us to create a CNAME record so they can verify domain ownership. To do that, use ```aws acm describe-certificate``` command to retrieve those values.

Store the result in a variable to prepare for extracting name and value later.

```shell
$ SSL_CERT_JSON=`aws acm describe-certificate \
--certificate-arn $SSL_CERT_ARN \
--query Certificate.DomainValidationOptions \
--region us-east-1`
```

Extract name and value.
```shell
$ SSL_CERT_NAME=`echo $SSL_CERT_JSON \
| jq -r ".[] | select(.DomainName == \"$DOMAIN_NAME\").ResourceRecord.Name"`
```

```shell
$ SSL_CERT_VALUE=`echo $SSL_CERT_JSON \
| jq -r ".[] | select(.DomainName == \"$DOMAIN_NAME\").ResourceRecord.Value"`
```

Verify name and value have been captured.
```shell
$ echo $SSL_CERT_NAME
_3f88376edb1eda680bd44991197xxxxx.abelperez.info.
```
```shell
$ echo $SSL_CERT_VALUE
_f528dff0e3e6cd0b637169a885xxxxxx.acm-validations.aws.
```

At this point, we are ready to interact with Route 53 to create the record set using the proposed values from ACM.

Get the hosted zone id, it can be copied from the console, but we can get it from route53 command line filtering by domain name.

```shell
$ R53_HOSTED_ZONE=`aws route53 list-hosted-zones-by-name \
--dns-name $DOMAIN_NAME \
--query HostedZones \
| jq -r ".[] | select(.Name == \"$DOMAIN_NAME.\").Id" \
| sed 's/\/hostedzone\///'`
```

Route 53 gives us the hosted zone id in the form of "/hostedzone/Z2TXYZQWVABDCE", the leading "/hostedzone/" bit is stripped out using ```sed``` command. 

Verify the hosted zone is captured in the variable.
```shell
$ echo $R53_HOSTED_ZONE
Z2TXYZQWVABDCE
```

With the **hosted zone id**, **name** and **value** from ACM, prepare the JSON input for route 53 ```change-resource-record-sets``` command, in this is case, Action is a CREATE, TTL can be the default 300 seconds (which is what AWS does itself through the console)

```shell
$ read -r -d '' R53_CNAME_JSON << EOM
{
  "Comment": "DNS Validation CNAME record",
  "Changes": [
    {
      "Action": "CREATE",
      "ResourceRecordSet": {
        "Name": "$SSL_CERT_NAME",
        "Type": "CNAME",
        "TTL": 300,
        "ResourceRecords": [
          {
            "Value": "$SSL_CERT_VALUE"
          }
        ]
      }
    }
  ]
}
EOM
```

Verify all variables were expanded correctly.
```shell
$ echo "$R53_CNAME_JSON"
{
  "Comment": "DNS Validation CNAME record",
  "Changes": [
    {
      "Action": "CREATE",
      "ResourceRecordSet": {
        "Name": "_3f88376edb1eda680bd44991197xxxxx.abelperez.info.",
        "Type": "CNAME",
        "TTL": 300,
        "ResourceRecords": [
          {
            "Value": "_f528dff0e3e6cd0b637169a885xxxxxx.acm-validations.aws."
          }
        ]
      }
    }
  ]
}
```

Create the record set using route 53.

```shell
$ R53_CNAME_ID=`aws route53 change-resource-record-sets \
--hosted-zone-id $R53_HOSTED_ZONE \
--change-batch "$R53_CNAME_JSON" \
--query ChangeInfo.Id \
--output text`
```

This operation will return a change-id, since route 53 needs to propagate the change, it won't be available immediately, usually within 60 seconds, to ensure we can proceed, use the wait command. This command will block the console until the record set change is ready.

```shell
$ aws route53 wait resource-record-sets-changed --id $R53_CNAME_ID
```

At this point, the record set is ready, now ACM needs to validate it, as per AWS docs, it can take up to several hours but in my experience it's not that long. By using another wait command, we'll block the UI until the certificate is validated.

```shell
$ aws acm wait certificate-validated \
--certificate-arn $SSL_CERT_ARN \
--region us-east-1
```

Once this is done, verify that it's in fact issued.
```shell
$ aws acm describe-certificate \
--certificate-arn $SSL_CERT_ARN \
--query Certificate.Status \
--region us-east-1
"ISSUED"
```

## Deploy the monolithic template

With the SSL certificate already created and stored in variable $SSL_CERT_ARN, let's define the other variables as input of infrastructure stack. This template can be deployed in any region provided all the services are available on that region.

In this case, IncludeRedirectToSubDomain parameter is set to true, this is useful when defining www as sub domain as a redirect rule will be created from abelperez.info to www.abelperez.info.

```shell
$ DOMAIN_NAME=abelperez.info
$ SUB_DOMAIN_NAME=www
$ INFRA_STACK_NAME=abelperez-info-infra
```

Deploy the infrastructure stack using the variables previously defined, it can take up to 40 minutes because of the CloudFront distribution that needs to propagate.

```shell
$ aws cloudformation deploy --stack-name $INFRA_STACK_NAME \
--template-file infrastructure.yaml \
--capabilities CAPABILITY_IAM CAPABILITY_NAMED_IAM \
--parameter-overrides \
DomainName=$DOMAIN_NAME \
SSLCertificate=$SSL_CERT_ARN \
SubDomainName=$SUB_DOMAIN_NAME \
IncludeRedirectToSubDomain=true
```
```
Waiting for changeset to be created..
Waiting for stack create/update to complete
Successfully created/updated stack - abelperez-info-infra
```

## Resources configuration

Once this stack is deployed, all resource are provisioned but there are some steps that need to be done in order to be fully operational with these resources.

Outputs from infrastructure stack will be needed in next steps. Store the JSON in a variable for later extraction.

```shell
$ INFRA_STACK_OUTPUTS=`aws cloudformation describe-stacks \
--stack-name $INFRA_STACK_NAME \
--query Stacks[0].Outputs`
```

Extract CodeCommit repository clone url via SSH.

```shell
$ CC_SSH_URL=`echo $INFRA_STACK_OUTPUTS \
| jq -r '.[] | select(.OutputKey == "CodeCommitRepositoryCloneUrlSsh").OutputValue'`
```

Extract S3 bucket name to store static files.

```shell
$ S3_BUCKET_NAME=`echo $INFRA_STACK_OUTPUTS \
| jq -r '.[] | select(.OutputKey == "S3StaticBucketName").OutputValue'`
```

Extract CodeCommit user name to associate with an SSH key.

```shell
$ CC_USER_NAME=`echo $INFRA_STACK_OUTPUTS \
| jq -r '.[] | select(.OutputKey == "CodeCommitUserName").OutputValue'`
```

### Upload SSH key to CodeCommit user

One of the ways to connect to CodeCommit git repository is via SSH key like any other git repository such as GitHub. In the next step we'll create a new SSH key only for this purpose. 

Set the file name in default location.

```shell
$ SSH_KEY_FILE=~/.ssh/aws_cc_rsa
```

Create the SSH key, replace your name.

```shell
$ ssh-keygen -t rsa -b 4096 -C "Your name here" -f $SSH_KEY_FILE -N ""
```

This might be needed (double check)
```shell
$ eval "$(ssh-agent -s)"
$ ssh-add $SSH_KEY_FILE
```

Upload the public key to IAM by using the ```upload-ssh-public-key``` command, the convention is the same file name with the .pub extension for public key.

```shell
$ CC_SSH_KEY_ID=`aws iam upload-ssh-public-key --user-name $CC_USER_NAME \
--ssh-public-key-body "$(cat $SSH_KEY_FILE.pub)" \
--query SSHPublicKey.SSHPublicKeyId \
--output text`
```

This gives us the user id associated with that public key in IAM. It should look something like this:

```shell
$ echo $CC_SSH_KEY_ID
APK0123456789ABCDEFG
```

### Configure SSH host names

The next step is optional but I recommend it in order to keep SSH configuration tidy. Taken from [this gist](https://gist.github.com/justinpawela/3a7056cd592d688425e59de2ef6f1da0) it's particularly helpful when we have several code commit repositories with different user accounts.

Extract the host name from CodeCommit clone url by using some ```sed``` substitutions. 

```shell
$ CC_SSH_HOST=`echo $CC_SSH_URL | sed 's/ssh\:\/\///' | sed 's/\/.*//'`
```

The host name will be something like this:

```shell
$ echo $CC_SSH_HOST
git-codecommit.eu-west-1.amazonaws.com
```

Choose an alias for that host name

```shell
$ CC_SSH_VHOST=awscodecommit-user1
```

Prepare the configuration settings block to add to ~/.ssh/config file. Note the blank lines before and after this block, this is to avoid conflicts with previous content in that file.

```shell
$ cat >>~/.ssh/config <<EOF

#
# Credentials for Account1
#
# '$CC_SSH_VHOST' is a name you pick
# '$CC_SSH_HOST' points to CodeCommit in the '$(aws configure get default.region)' region
# '$CC_SSH_KEY_ID' UserID as provided by IAM Security Credentials (SSH)
# '$SSH_KEY_FILE' Path to corresponding key file

Host $CC_SSH_VHOST
  Hostname $CC_SSH_HOST
  User $CC_SSH_KEY_ID
  IdentityFile $SSH_KEY_FILE

EOF
```

### Clone the respository

To keep consistency with the configuration above, clone url has to be updated with the chosen virtual host name earlier.

```shell
$ CC_SSH_CLONE_URL=`echo $CC_SSH_URL | sed "s/$CC_SSH_HOST/$CC_SSH_VHOST/"`
```

Check the url clone url.

```shell
$ echo $CC_SSH_CLONE_URL
ssh://awscodecommit-abelperez-info/v1/repos/www.abelperez.info-web
```

Choose a local directory to clone the repository.

```shell
$ CC_LOCAL_REPO=~/Desktop/repo
```

Clone the repository.

```shell
$ git clone $CC_SSH_CLONE_URL $CC_LOCAL_REPO
Cloning into '/home/abel/Desktop/repo'...
warning: You appear to have cloned an empty repository.
```

### Put some content in the repository

Assuming current directory is still ```templates/v1```, copy sample s3 bucket content to local repository and move to repository directory.

```shell
$ cp -r ../../s3-bucket-content/* $CC_LOCAL_REPO
$ cd $CC_LOCAL_REPO
```

The sample content consists of a blank HTML file displaying a Hello World message and a basic template for CodeBuild (buildspec.yml) where there is a placeholder for the s3 bucket name which will depend on the stack's output that was captured earlier in variable S3_BUCKET_NAME.

```shell
$ echo $S3_BUCKET_NAME
abelperez-info-infra-staticsitebucket-x6e3g8pqm6au
```

Replace placeholder with actual bucket name.

```shell
$ sed -i "s/<INSERT-YOUR-BUCKET-NAME-HERE>/$S3_BUCKET_NAME/" buildspec.yml
```

Verify the variable was expanded correctly.

```shell
$ cat buildspec.yml
version: 0.2

phases:
  build:
    commands:
      - mkdir dist
      - cp *.html dist/

  post_build:
    commands:
      - aws s3 sync ./dist s3://abelperezmartinez-com-infra-staticsitebucket-x6e3g8pqm6au/ --delete --acl=public-read
```

## Start using the repository