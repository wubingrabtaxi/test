## Dockerizer Tool For Project Food

[![N|Solid](https://www.onlinebooksreview.com/uploads/blog_images/2017/10/29_docker_twitter_share.png)](https://nodesource.com/products/nsolid)

Dockerizer helps dockerize a golang project
  - Create repo in aws ECR on Food Subaccount
  - Build/Upload project docker images
  - Runs project docker images locally

# Prerequisites
  - aws python sdk
  - aws access key and secret for food subaccount
  - docker for osx

## Setup docker for osx
https://docs.docker.com/docker-for-mac/install/

## AWS SDK
let's install virtualenv first:
https://virtualenv.pypa.io/en/stable/installation/
Once you have virtualenv installed, create a new virtualenv called aws
```sh
virtualenv create aws
source aws/bin/activate
#when you want to quit the virtualenv, issue following command:
deactivate
```
now let's assume you are in aws virtualenv (activated):
```sh
pip install awscli
pip install boto3
```
you should be able to install both hassle free, now let's configure aws cli access:
https://docs.aws.amazon.com/cli/latest/userguide/cli-chap-getting-started.html
```sh
aws configure
```
Plase make sure you enter `ap-southeast-1` when prompted for default zone.
and `json` for default output format.

Congrats! You are set for AWS sdk setup.

## Setup your project
for 1 project, there should 1 corresponding dockerizer, because we will use setup.py to setup the context for build/upload/run.
```sh
cd food-dockerizer
python setup.py
```
^ is one time only setup. After this step is done, we could start build/upload/run images of your projects.
```sh
./stg.sh build upload #this will build the staging image, upload it to ECR
./prd.sh build upload #this will build the production image, upload it to ECR
```


