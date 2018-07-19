# Serverless Static Web Infrastructure

This repository contains a set of AWS CloudFormation templates to automate the creation of a serverless static website including the whole end to end process as explained in [this guide](https://abelperezmartinez.blogspot.com/2018/04/completely-serverless-static-website-on-aws.html).

## How this helps you

You want to start a serverless static website i.e. your personal website and you want to host it on AWS but there seems to be a lot work involved, many steps, many different services to manage and set up.

You also want to use version control and some form of continuous integration, ideally fully automated from pushing code to remote repository to see the new version completely deployed.

By using these templates, you'll get the following ready to use:

* SSL certificate 
* S3 static Web hosting 
* CloudFront distribution
* Route53 record set 
* CodeCommit repository (including user and role)
* CodeBuild project (including associated role)
* CloudWatch event rule (to connect repo with build)

## Prerequisites

* AWS account
* Route 53 - Hosted zone for the given domain
* AWS CLI configured with a default profile (or custom)
* jq installed i.e ```sudo apt-get install jq```

## Folder structure

* **s3-bucket-content** - sample HTML page and buildspec.yml file
* **templates/v1** - monolithic template (development pipeline and web infra in the same template) and using double CloudFront and double S3 bucket for non-www to www redirection.
* **templates/v2** - refactored templates (development pipeline and web infra in separate templates) and using Lambda@Edge instead of double CloudFront and double S3 bucket for non-www to www redirection.

## Input parameters

### General parameters

* DomainName - The site domain name (naked domain only), this is used to create the SSL certificate for the given domain and sub domains as Subject Alternative Names field value.
* SubDomainName - The site sub domain name (sub domain only), this is used to create DNS entry in Route 53, typical values are www, blog, etc.
* IncludeRedirectToSubDomain - Determines whether we want to create an automatic redirection rule from the naked domain to specified sub domain, typically is used to create a redirect from abelperez.info to www.abelperez.info totally automatic.

### Stack names

Each CloudFormation stack requires a name, this name will be used to compose all the resources such as bucket names, code repositories, etc.

### Other variables

* SSH Key file name - I chose to use SSH Key to authenticate with Git, for which I suggest to create a new SSH key but you can reuse an existing one if you wish so.
* CodeCommit alias hostname - I chose to use an alias for each CodeCommit repository host name to avoid confusing IAM user ID and the corresponding SSH key, you can use the default host name provided by AWS if you don't have more than one user and CodeCommit repository in the same region.
* Local git repo location - Typically a directory inside your local home directory i.e ~/Desktop/repo.

## How to use [version 1 templates](templates/v1/README.md)

## How to use [version 2 templates](templates/v2/README.md)