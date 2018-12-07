# Alfalfa

This is a Haystack implementation backed by a virtual building. Virtual building simulation is performed using OpenStudio and EnergyPlus.

## Getting Started

1. Install [Docker](https://www.docker.com) for your platform.
1. From a command prompt ```docker-compose up web```.
1. Navigate to http://localhost/api/nav to verify that the Haystack implementation is running.
1. Navigate to http://localhost to view web application.
1. Navigate to http://localhost:9000 to view the minio file server.
This is a local implementation of Amazon S3 for bulk file storage during development.
1. Use ```Ctrl-C``` to stop the services.

## Running Worker Tests

1. Run ```docker-compose up worker-test``` 
1. Test output should be located in worker/test/output. 
1. See worker/test/test.py for an example. 

## Making changes

1. Make source code changes as needed.
1. Verify that the Docker services are stopped with ```docker-compose down```.
1. Recreate the containers and restart services with ```docker-compose up```.

This is a basic workflow that is sure to work, other docker and docker-compose commands can be used to start / stop services without rebuilding. 

## Project Organization 

Development of this project is setup to happen in a handful of docker containers. This makes it easy to do ```docker-compose up```, and get a fully working development environment.  There are currently separate containers for web (The Haystack API server), worker (The thing that runs the simulation, aka master algorithm), database (mongo, the thing that holds a model's point dictionary and current / historical values), and queue (The thing where API requests go when they need to trigger something in the master algorithm). In this project there is one top level directory for each docker container.

## Design

https://docs.google.com/presentation/d/1fYtwXvS4jlrhTdHJxPUPpbZZ8PwIM8sGCdLsG_6hwwo/edit#slide=id.p

## Deployment

1. Set environment variables, WEB_REGISTRY_URI, and WORKER_REGISTRY_URI.
These variables are only used for cloud deployment, but docker-compose will require a value
```
export WEB_REGISTRY_URI=313781303390.dkr.ecr.us-east-1.amazonaws.com/queue/web 
export WORKER_REGISTRY_URI=313781303390.dkr.ecr.us-east-1.amazonaws.com/queue/worker
```
1. Export the NODE_ENV variable
```
export NODE_ENV="production"
```

## Create a new AWS account

1. Navigate to the IAM dashboard https://console.aws.amazon.com/iam/home
1. Click "Users" -> "Add User"
1. Enter a username such as "batman"
1. Select "Programmatic access" and "AWS Management Console access" for access type
1. Accept the default option "Autogenerated password" and "Requir password reset"

### Give user permissions

1. Continuing from the new AWS account sequence, choose "Create Group"
1. Assing a group name, such as "superhero"
1. Select the "AdministratorAccess" policy
1. Click "Greate Group"

### Review

1. From the user permissions sequence, choose "Next: Review"
1. Click "Create User"
1. Click "Close"

## Create an access key for new user

1. Navigate to the IAM dashboard https://console.aws.amazon.com/iam/home
1. Navigate to Users -> batman -> security credentials
1. Click "Create Access Key"
1. Save the public and secret keys

## Install and configure the AWS cli

1. Follow [these](http://docs.aws.amazon.com/cli/latest/userguide/installing.html) directions for cli installation
1. From a command prompt enter: ```aws configure```
1. Enter the public and secret keys created previously when prompted
1. Enter "us-east-1" for the region
1. Accept the default "None" for output, which will fallback to Json format

## Create an S3 bucket 

1. aws s3 mb s3://acmesafe
1. Edit the runBRICR.py script to push to the s3 bucket. Bucket names are unique in a region, therefore
you have to pick a unique name. The is not currently a configuration mechanism for this so you have to edit the 
source. TODO: fix that.

## Create a virtual private cloud

This step is probably already done for a new aws account, but if not the following steps 
come from here: https://docs.aws.amazon.com/AmazonVPC/latest/UserGuide/vpc-subnets-commands-example.html

1. Create the vpc and record the VpcID that is returned ```
aws ec2 create-vpc --cidr-block 10.0.0.0/16 ```
1. Create two new subnets in the VPC ```
aws ec2 create-subnet --vpc-id vpc-2cc7ee54 --cidr-block 10.0.1.0/24 --availability-zone us-east-1a
aws ec2 create-subnet --vpc-id vpc-2cc7ee54  --cidr-block 10.0.0.0/24 --availability-zone us-east-1b```
1. Create a new internet gateway and record the InternetGatewayId ```
aws ec2 create-internet-gateway ```
1. Attach the new internet gateway to the vpc ```
aws ec2 attach-internet-gateway --vpc-id vpc-2cc7ee54 --internet-gateway-id igw-6eb79017```
1. Lookup the route table id using the aws console. A default table is created with the vpc.
Create a route in the route table that points all traffic (0.0.0.0/0) to the Internet gateway. ```
aws ec2 create-route --route-table-id rtb-be8522c3 --destination-cidr-block 0.0.0.0/0 --gateway-id igw-6eb79017```

## Add a rule to the default group to open port 80

```aws  ec2 authorize-security-group-ingress --group-name default --protocol tcp --port 80 --cidr 0.0.0.0/0```
or if there are multiple default security gropus, then use the --group-id option
```aws  ec2 authorize-security-group-ingress --group-id sg-xyz --protocol tcp --port 80 --cidr 0.0.0.0/0```

## Create a load balancer

https://docs.aws.amazon.com/elasticloadbalancing/latest/application/tutorial-application-load-balancer-cli.html

```
aws elbv2 create-load-balancer --name alfalfa-load-balancer --subnets subnet-01acff4a subnet-4b1a8f64 --security-groups sg-766ab202
aws elbv2 create-target-group --name alfalfa-web-targets --protocol HTTP --port 80 --vpc-id vpc-2cc7ee54 --target-type ip
aws elbv2 create-listener --load-balancer-arn arn:aws:elasticloadbalancing:us-east-1:431265468707:loadbalancer/app/alfalfa-load-balancer/81957ac367b321fb --protocol HTTP --port 80 --default-actions Type=forward,TargetGroupArn=arn:aws:elasticloadbalancing:us-east-1:431265468707:targetgroup/alfalfa-web-targets/9b10c67fcdd458a4
```

## Create new container registries 

Create two new container registries, one to hold the "web" app container,
and another to hold the "worker" app container.

1. aws ecr create-repository --repository-name queue/web
1. aws ecr create-repository --repository-name queue/worker
1. Note the repository arn that is returned from the previous commands, ie: ```
  {
    "repository": {
        "registryId": "313781303390",
        "repositoryName": "queue/web",
        "repositoryArn": "arn:aws:ecr:us-east-1:313781303390:repository/queue/web",
        "createdAt": 1513363732.0,
        "repositoryUri": "313781303390.dkr.ecr.us-east-1.amazonaws.com/queue/web"
    }
}```
1. Set environment variables, WEB_REGISTRY_URI, and WORKER_REGISTRY_URI, ie: ```
export WEB_REGISTRY_URI=313781303390.dkr.ecr.us-east-1.amazonaws.com/queue/web
export WORKER_REGISTRY_URI=313781303390.dkr.ecr.us-east-1.amazonaws.com/queue/worker```
1. Be sure these variables are set during all of the following steps,
especially dock-compose build, and while using the deploy script.

## Setup a mongo database with mlab

1. Follow mlab's guided process to make a new account
1. After the account is acive, complete the next steps from the home page, https://mlab.com/home
1. From "MongoDB Deployments" on the home page click "Create New"
1. Choose AWS as the cloud provider, and select the free sandbox option
1. Click continue
1. Select the us-east-1 region
1. Enter "walnut" as the database name
1. "Submit Order"
1. After the database is create select it from the interface and record the uri, ie: ```
mongodb://<dbuser>:<dbpassword>@ds141786.mlab.com:41786/walnut```
1. From the "Users" interface (https://mlab.com/databases/walnut#users), select "Add database user"
1. Enter username "batman", choose a password, and remember it
1. Set the environment variable, MONGO_URL, replacing dbuser and dbpassword, with batman and the corresponding password ```
export MONGO_URL=mongodb://<dbuser>:<dbpassword>@ds141786.mlab.com:41786/walnut```

## Setup an aws SQS queue

1. Use the aws cli to create a new queue named pine ```
aws sqs create-queue --queue-name pine```
1. Record the queueUrl that is returned, ie: ```
{
    "QueueUrl": "https://queue.amazonaws.com/313781303390/pine"
}```
1. Set the environment variable, JOB_QUEUE_URL, ie: ```
export JOB_QUEUE_URL=https://queue.amazonaws.com/313781303390/pine```

## Push to container registries

1. Install docker, https://www.docker.com/community-edition#/download
1. Use ```aws ecr get-login | bash -``` to login to docker.
1. Build current containers with, ```docker-compose build```.
1. Use ```docker-compose push``` to push new images to the aws container registry.

## Create an ecs cluster

1. Create an ECS cluster to attach the new EC2 instances to ```
aws ecs create-cluster --cluster-name worker_ecs_cluster```

## Create IAM Roles

1. Create an IAM Role for the worker service ```
aws iam create-role --role-name Worker-Role --assume-role-policy-document file://<porject-root>/deploy/Worker-Role-Trust-Policy.json```
1. Record the arn that is returned, ie```
"Arn": "arn:aws:iam::313781303390:role/Worker-Role"```
1. Export the arn as an environment variable ```
export WORKER_ROLE_ARN=arn:aws:iam::313781303390:role/Worker-Role```
1. Attach the required policies to the new role ```
aws iam attach-role-policy --role-name Worker-Role --policy-arn arn:aws:iam::aws:policy/AmazonSQSFullAccess
aws iam attach-role-policy --role-name Worker-Role --policy-arn arn:aws:iam::aws:policy/AmazonS3FullAccess
aws iam attach-role-policy --role-name Worker-Role --policy-arn arn:aws:iam::aws:policy/service-role/AmazonEC2ContainerServiceforEC2Role```
aws iam create-policy --policy-name ServiceScaling --policy-document file:///Users/kbenne/Development/alfalfa/deploy/update-service-policy.json
aws iam attach-role-policy --role-name Worker-Role --policy-arn <insert arn returned from previous command>```
1. Create an IAM Role to execute the tasks ```
aws iam create-role --role-name Execution-Role --assume-role-policy-document file:///<project-root>/deploy/Instance-Role-Trust-Policy.json```
1. Record the arn that is returned, ie```
"Arn": "arn:aws:iam::431265468707:role/Execution-Role"```
1. Export the arn as an environment variable ```
export EXECUTION_ROLE_ARN=arn:aws:iam::431265468707:role/Execution-Role```
1. Attach the required policies to the new role ```
aws iam attach-role-policy --role-name Execution-Role --policy-arn arn:aws:iam::aws:policy/service-role/AmazonEC2ContainerServiceforEC2Role```

## Create a container service for the worker

1. Register the worker task definition using the deploy script. It is important that all of the environment variables are set at this point```
./deploy/deploy create -s worker```
1. Create the worker service. One EC2 instance should fire up after this```
aws ecs create-service --service-name worker-service --launch-type "FARGATE" --network-configuration "awsvpcConfiguration={subnets=[subnet-01acff4a,subnet-4b1a8f64], assignPublicIp=ENABLED, securityGroups=[sg-766ab202]}"  --task-definition worker --cluster worker_ecs_cluster --desired-count 1```

## Create a container service for the web

1. Register the worker task definition using the deploy script. It is important that all of the environment variables are set at this point```
./deploy/deploy create -s web```
```aws ecs create-service --service-name web-service --launch-type "FARGATE" --network-configuration "awsvpcConfiguration={subnets=[subnet-01acff4a,subnet-4b1a8f64], assignPublicIp=ENABLED, securityGroups=[sg-766ab202]}"  --task-definition web --cluster worker_ecs_cluster --desired-count 1```

## Enable scaling for the worker service

1. Make a scalable target```
aws application-autoscaling  register-scalable-target --service-namespace ecs --resource-id service/worker_ecs_cluster/worker-service --scalable-dimension ecs:service:DesiredCount --min-capacity 1 --max-capacity 200```
1. Make a scale out policy for the worker service, record the PolicyARN that is returned```
aws application-autoscaling  put-scaling-policy --cli-input-json file:///<project-root>deploy/Worker-Service-Scale-Out-Policy.json```
1. Create a scale out alarm for the worker service```
aws cloudwatch put-metric-alarm --alarm-name WorkerServiceScaleOutAlarm --metric-name ApproximateNumberOfMessagesVisible --namespace AWS/SQS --statistic Average --unit Count --comparison-operator GreaterThanOrEqualToThreshold --threshold 1 --period 60 --evaluation-periods 1 --dimensions Name=QueueName,Value=pine   --alarm-actions <copy scaling policy arn>

## If you need to update the worker

1. Install docker, https://www.docker.com/community-edition#/download
1. Use ```aws ecr get-login | bash -``` to login to docker.
1. Build current containers with, ```docker-compose build```.
1. Use ```docker-compose push``` to push new images to the aws container registry.
1. ```./deploy/deploy create```

