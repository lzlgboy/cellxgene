# AWS Elastic Beanstalk 

This directory contains script to aid in creating and deploying cellxgene on
an AWS Elastic Beanstalk instance.

This will result in a variant of cellxgene, running on AWS EC2 instances, serving data from S3. 
All datasets must be in the new CXG (tiledb) format - see the converter script cxgtool.py 
in server/converters - and located in a single S3 prefix, which is accessible to the instance. 
In the current incarnation, no access control or authentication support is available 
(outside of anything you configure yourself), so this is most appropriate for public datasets.

This is early development work, and will change significantly in the near future. 
We would love feedback on it, but please assume it will change.

## Prerequisites

1. Some familiarity with AWS EB, S3, and IAM are needed.

2. Install the awsebcli.
Instruction are here:  
https://docs.aws.amazon.com/elasticbeanstalk/latest/dg/eb-cli3-install.html

3. In the top level directory, run ```make build-client``` to create the client static assets. 

## Steps

These steps are meant to serve as an example.  
There are many more options to these commands that may be important or necessary for your environment.

1. Create an S3 bucket

   Upload your matrix files to this bucket
   
2. Create an elastic beanstalk application.  For example:

    ```
    EB_APP=cellxgene-app
    eb init -p python-3.6 $EB_APP
    ```

3. Configuring cellxgene

   All the cellxgene configuration options can be set from a configuration file.
   This file can be generated like this:
   
   ```cellxgene launch --dump-default-config > myconfig.yaml```
   
   The config file may then be customized before the app is deployed.
   
   If your config file is named "config.yaml" and exists in this directory, then it will be bundled with the 
   application zip file and installed along side the app on the EB servers.
   
   However, a potentially more flexible approach is to place your config file in a location accessible to the EB 
   servers, such as along side the matrix files in S3.  For example:  s3://my-bucket/my-datasets/config.yaml.
   Set the CXG_CONFIG_FILE environment variable to specify this location.  
   
   Another option is to set the CXG_DATAROOT environment variable.  The dataroot 
   is the location where the matrix files are located. 
   This environment variable will override the dataroot in the config file (if specified).
   
   - Note:  Certain features, such as diffexp and user annotations, are automatically disabled by the EB app,
   and cannot be enabled using configuration.  They may be enabled manually by modifying app.py, however
   this is not supported or recommended at this time.
   
4. Prepare static endpoints

   The cellxgene server can server additional static webpages that may be associated with the deployment.
   These include the about_legal_tos (terms of service), and bout_legal_privacy, for example.  
   To use this feature, do the following:

   * In this directory, create a sub directory called "static".  
   * Copy the files you want to server into this directory
   * modify your configuration file to set the location to these file:  /static/deploy/<filename>

   Example:  you want to include an "about_legal_tos" and "about_legal_privacy" page to cellxgene.
   Assume files called "tos.html" and "privacy.html" exist.

   ```
   $ mkdir static
   $ cp <source_dir>/tos.html static/tos.html
   $ cp <source_dir>/privacy.html static/privacy.html

   # edit config.yaml
   $ grep "/static/deploy" config.yaml
   about_legal_tos: /static/deploy/tos.html
   about_legal_privacy: /static/deploy/privacy.html
   ```
   
   The next step will place these files into the deployment.
   
5. Create the artifact.zip file for the application

   ```
   $ make build
   ```
    
6. Flask secret key

   The application requires as secret key to be provided to flask, the web framework used by cellxgene.
   There are three ways to provide the secret key:
   
   - In the configuration file:  update the server/flask_secret_key attribute.
   - An environment variable:  CXG_SECRET_KEY
   - Managed by the AWS Secret Manager
   
   If using the AWS Secret Manager, then the secret name is passed as an environment variable:  CXG_AWS_SECRET_NAME.
   The secret must contain a key with the name "flask_secret_key".
   Likely you have located the AWS Secret Manager in the same AWS region as the dataroot.  If that is not the case
   then the AWS Secret Manager region name can be specified in an environment variable:  CXG_AWS_SECRET_REGION_NAME.
   
   
7. Create an environment

    ```
    # name of the environment
    $ EB_ENV=cellxgene-env
   
    # type of ec2 instance to run the cellxgene server (for example)
    $ EB_INSTANCE=m5.large 
   
    # One or both of the following environment variables needs to be set
    $ CXG_DATAROOT=<location to your S3 bucket>
    $ CXG_CONFIG_FILE=<location to your config file>
  
    # Potentially also set envvars for the sercret key.
   
    $ eb create $EB_ENV --instance-type $EB_INSTANCE \
       --envvars CXG_DATAROOT=$CXG_DATAROOT,CXG_CONFIG_FILE=$CXG_CONFIG_FILE
    ```

8. Give the elastic beanstalk environment access to the S3 bucket.

    This link may provide some useful information:
    https://aws.amazon.com/premiumsupport/knowledge-center/elastic-beanstalk-s3-bucket-instance/
    
9. Deploy the application

    ```
    $ eb deploy $EB_ENV 
    ```
   
10. Open the application in a browser

   ```
   $ eb open $EB_ENV
   ```
