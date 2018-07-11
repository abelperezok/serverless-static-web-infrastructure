# Serverless Static Web Infrastructure

This repository contains a set of AWS CloudFormation templates to automate the creation of a serverless static website including the whole end to end process as explained in [this guide](https://abelperezmartinez.blogspot.com/2018/04/completely-serverless-static-website-on-aws.html).

## Folder structure

* s3-bucket-content - sample HTML page and buildspec.yml file
* templates/v1 - monolithic template (development pipeline and web infra in the same template) and using double CloudFront and double S3 bucket for non-www to www redirection.
* templates/v2 - refactored templates (development pipeline and web infra in separate templates) and using Lambda@Edge instead of double CloudFront and double S3 bucket for non-www to www redirection.

## AWS resources created - v1

After creating stacks based on these templates you'll have in your account the following resources:

* AWS::CertificateManager::Certificate (US-EAST-1)
* 2 x AWS::S3::Bucket (if not using redirection, only 1)
* 2 x AWS::CloudFront::Distribution (if not using redirection, only 1)
* 2 x AWS::Route53::RecordSet (if not using redirection, only 1)
* AWS::CodeCommit::Repository 
* AWS::IAM::User (user to interact with git)
* 2 x AWS::IAM::Role (CodeBuild project and CloudWatch event rule)
* 3 x AWS::IAM::ManagedPolicy (user and the two roles)
* AWS::CodeBuild::Project
* AWS::Events::Rule

## AWS resources created - v2

After creating stacks based on these templates you'll have in your account the following resources:

### Base infrastructure (US-EAST-1 Edge network)
* AWS::CertificateManager::Certificate 
* AWS::IAM::ManagedPolicy 
* AWS::IAM::Role
* AWS::Lambda::Function
* AWS::Lambda::Version

### Web infrastructure
* AWS::S3::Bucket
* AWS::CloudFront::Distribution
* 2 x AWS::Route53::RecordSet (if not using redirection, only 1)

### Build pipeline infrastructure
* AWS::CodeCommit::Repository
* AWS::IAM::User
* 2 x AWS::IAM::Role
* 3 x AWS::IAM::ManagedPolicy
* AWS::CodeBuild::Project
* AWS::Events::Rule
