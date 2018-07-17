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

if validating certificate using Email, it can be created by base template:

commands ...


if validating certificate using DNS, it has to created via cli and provide the certificate ARN to base template.

commands ...

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
