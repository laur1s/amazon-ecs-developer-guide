# Tutorial: Creating a Cluster with an EC2 Task Using the Amazon ECS CLI<a name="ecs-cli-tutorial-ec2"></a>

This tutorial shows you how to set up a cluster and deploy a task using the EC2 launch type\.

## Prerequisites<a name="ECS_CLI_EC2_prerequisites"></a>

Complete the following prerequisites:
+ Set up an AWS account\.
+ Install the Amazon ECS CLI\. For more information, see [Installing the Amazon ECS CLI](ECS_CLI_installation.md)\.
+ Install and configure the AWS CLI\. For more information, see [AWS Command Line Interface](https://docs.aws.amazon.com/cli/latest/userguide/cli-environment.html)\.

## Step 1: Configure the Amazon ECS CLI<a name="ECS_CLI_tutorial_configure"></a>

Before you can start this tutorial, you must install and configure the Amazon ECS CLI\. For more information, see [Installing the Amazon ECS CLI](ECS_CLI_installation.md)\.

The Amazon ECS CLI requires credentials in order to make API requests on your behalf\. It can pull credentials from environment variables, an AWS profile, or an Amazon ECS profile\. For more information, see [Configuring the Amazon ECS CLI](ECS_CLI_Configuration.md)\.

**To create an Amazon ECS CLI configuration**

1. Create a cluster configuration:

   ```
   ecs-cli configure --cluster ec2-tutorial --region us-east-1 --default-launch-type EC2 --config-name ec2-tutorial
   ```

1. Create a profile using your access key and secret key:

   ```
   ecs-cli configure profile --access-key AWS_ACCESS_KEY_ID --secret-key AWS_SECRET_ACCESS_KEY --profile-name tutorial-profile
   ```
**Note**  
If this is the first time that you are configuring the Amazon ECS CLI, these configurations are marked as default\. If this is not your first time configuring the Amazon ECS CLI, see [ecs\-cli configure default](cmd-ecs-cli-configure-default.md) and [ecs\-cli configure profile default](cmd-ecs-cli-configure-profile-default.md) to set this as the default configuration and profile\.

## Step 2: Create Your Cluster<a name="ECS_CLI_tutorial_cluster_create"></a>

The first action you should take is to create a cluster of Amazon ECS container instances that you can launch your containers on with the ecs\-cli up command\. There are many options that you can choose to configure your cluster with this command, but most of them are optional\. In this example, you create a simple cluster of two `t2.medium` container instances that use the *id\_rsa* key pair for SSH access \(substitute your own key pair here\)\.

By default, the security group created for your container instances opens port 80 for inbound traffic\. You can use the `--port` option to specify a different port to open, or if you have more complicated security group requirements, you can specify an existing security group to use with the `--security-group` option\.

```
ecs-cli up --keypair id_rsa --capability-iam --size 2 --instance-type t2.medium --cluster-config ec2-tutorial --ecs-profile tutorial-profile
```

This command may take a few minutes to complete as your resources are created\. Now that you have a cluster, you can create a Docker compose file and deploy it\.

## Step 3: Create a Compose File<a name="ECS_CLI_tutorial_compose_create"></a>

For this step, create a simple Docker compose file that creates a WordPress application consisting of a web server and a MySQL database\. At this time, the Amazon ECS CLI supports [Docker compose file syntax](https://docs.docker.com/compose/compose-file/#versioning) versions 1, 2, and 3\. Examples for both Docker Compose version 2 and 3 are provided\.

The following parameters are supported in Compose files for the Amazon ECS CLI:
+ `cap_add` \(not valid for tasks using the Fargate launch type\)
+ `cap_drop` \(not valid for tasks using the Fargate launch type\)
+ `command`
+ `cpu_shares`
**Note**  
If you're using the Compose version 3\.0 format, `cpu_shares` should be specified in the `ecs-params.yml` file\. For more information, see [Using Amazon ECS Parameters](cmd-ecs-cli-compose-ecsparams.md)\.
+ `devices` \(not valid for tasks using the Fargate launch type\)
+ `dns`
+ `dns_search`
+ `entrypoint`
+ `environment`: If an environment variable value isn't specified in the Compose file, but it exists in the shell environment, the shell environment variable value is passed to the task definition that is created for any associated tasks or services\.
**Important**  
We don't recommend using plaintext environment variables for sensitive information, such as credential data\.
+ `env_file`
**Important**  
We don't recommend using plaintext environment variables for sensitive information, such as credential data\.
+ `extends` \(Compose file version 1\.0 and 2 only\)
+ `extra_hosts`
+ `healthcheck` \(Compose file version 3\.0 only\)
**Note**  
The `start_period` field isn't supported using the Compose file\. To specify a `start_period`, use the `ecs-params.yml` file\. For more information, see [Using Amazon ECS Parameters](cmd-ecs-cli-compose-ecsparams.md)\.
+ `hostname`
+ `image`
+ `labels`
+ `links` \(not valid for tasks using the Fargate launch type\)
+ `log_driver` \(Compose file version 1\.0 only\)
+ `log_opt` \(Compose file version 1\.0 only\)
+ `logging` \(Compose file version 2\.0 and 3\.0\)
  + `driver`
  + `options`
+ `mem_limit` \(in bytes\)
**Note**  
If you're using the Compose version 3\.0 format, `mem_limit` should be specified in the `ecs-params.yml` file\. For more information, see [Using Amazon ECS Parameters](cmd-ecs-cli-compose-ecsparams.md)\.
+ `mem_reservation` \(in bytes\)
**Note**  
If you're using the Compose version 3\.0 format, `mem_reservation` should be specified in the `ecs-params.yml` file\. For more information, see [Using Amazon ECS Parameters](cmd-ecs-cli-compose-ecsparams.md)\.
+ `ports`
+ `privileged` \(not valid for tasks using the Fargate launch type\)
+ `read_only`
+ `security_opt`
+ `shm_size` \(Compose file version 1\.0 and 2 only and not valid for tasks using the Fargate launch type\)
+ `tmpfs` \(not valid for tasks using the Fargate launch type\)
+ `tty`
+ `ulimits`
+ `user`
+ `volumes`
+ `volumes_from` \(Compose file version 1\.0 and 2 only\)
+ `working_dir`

**Important**  
The `build` directive isn't supported at this time\.

For more information about Docker Compose file syntax, see the [Compose file reference](https://docs.docker.com/compose/compose-file/#/compose-file-reference) in the Docker documentation\.

Here is the compose file, which you can call `docker-compose.yml`\. Each container has 100 CPU units and 500 MiB of memory\. The `wordpress` container exposes port 80 to the container instance for inbound traffic to the web server\. A logging configuration for the containers is also defined\.

Example 1: Docker Compose version 2

```
version: '2'
services:
  wordpress:
    image: wordpress
    cpu_shares: 100
    mem_limit: 524288000
    ports:
      - "80:80"
    links:
      - mysql
    logging:
      driver: awslogs
      options: 
        awslogs-group: tutorial-wordpress
        awslogs-region: us-east-1
        awslogs-stream-prefix: wordpress
  mysql:
    image: mysql:5.7
    cpu_shares: 100
    mem_limit: 524288000
    environment:
      MYSQL_ROOT_PASSWORD: password
    logging:
      driver: awslogs
      options: 
        awslogs-group: tutorial-mysql
        awslogs-region: us-east-1
        awslogs-stream-prefix: mysql
```

Example 2: Docker Compose version 3

```
version: '3'
services:
  wordpress:
    image: wordpress
    ports:
      - "80:80"
    links:
      - mysql
    logging:
      driver: awslogs
      options: 
        awslogs-group: tutorial-wordpress
        awslogs-region: us-east-1
        awslogs-stream-prefix: wordpress
  mysql:
    image: mysql:5.7
    environment:
      MYSQL_ROOT_PASSWORD: password
    logging:
      driver: awslogs
      options: 
        awslogs-group: tutorial-mysql
        awslogs-region: us-east-1
        awslogs-stream-prefix: mysql
```

When using Docker Compose version 3 format, the CPU and memory specifications must be specified separately\. Create a file named `ecs-params.yml` with the following content:

```
version: 1
task_definition:
  services:
    wordpress:
      cpu_shares: 100
      mem_limit: 524288000
    mysql:
      cpu_shares: 100
      mem_limit: 524288000
```

## Step 4: Deploy the Compose File to a Cluster<a name="ECS_CLI_tutorial_compose_deploy"></a>

After you create the compose file, you can deploy it to your cluster with the ecs\-cli compose up command\. By default, the command looks for a compose file called `docker-compose.yml` and an optional ECS parameters file called `ecs-params.yml` in the current directory, but you can specify a different file with the `--file` option\. By default, the resources created by this command have the current directory in the title, but you can override that with the `--project-name project_name` option\. The `--create-log-groups` option creates the CloudWatch log groups for the container logs\.

```
ecs-cli compose up --create-log-groups --cluster-config ec2-tutorial --ecs-profile tutorial-profile
```

## Step 5: View the Running Containers on a Cluster<a name="ECS_CLI_tutorial_ps"></a>

After you deploy the compose file, you can view the containers that are running on your cluster with the ecs\-cli ps command\.

```
ecs-cli ps --cluster-config ec2-tutorial --ecs-profile tutorial-profile
```

Output:

```
Name                                                     State    Ports                      TaskDefinition  Health
ec2-tutorial/4a5c3162979b4df6b44d2f4e6b1dd3e5/wordpress  RUNNING  18.205.157.201:80->80/tcp  ecscompose:3    UNKNOWN
ec2-tutorial/4a5c3162979b4df6b44d2f4e6b1dd3e5/mysql      RUNNING                             ecscompose:3    UNKNOWN
```

In the above example, you can see the `wordpress` and `mysql` containers from your compose file, and also the IP address and port of the web server\. If you point a web browser to that address, you should see the WordPress installation wizard\.

## Step 6: Scale the Tasks on a Cluster<a name="ECS_CLI_tutorial_compose_scale"></a>

You can scale your task count up so you could have more instances of your application with the ecs\-cli compose scale command\. In this example, you can increase the count of your application to two\.

```
ecs-cli compose scale 2 --cluster-config ec2-tutorial --ecs-profile tutorial-profile
```

Now you should see two more containers in your cluster:

```
ecs-cli ps --cluster-config ec2-tutorial --ecs-profile tutorial-profile
```

Output:

```
Name                                            State    Ports                               TaskDefinition  Health
ec2-tutorial/4a5c3162979b4df6b44d2f4e6b1dd3e5/wordpress  RUNNING  18.205.157.201:80->80/tcp  ecscompose:3    UNKNOWN
ec2-tutorial/4a5c3162979b4df6b44d2f4e6b1dd3e5/mysql      RUNNING                             ecscompose:3    UNKNOWN
ec2-tutorial/da60d89e74474a2691d26ef5f9ba5a16/mysql      RUNNING                             ecscompose:3    UNKNOWN
ec2-tutorial/da60d89e74474a2691d26ef5f9ba5a16/wordpress  RUNNING  34.232.105.149:80->80/tcp  ecscompose:3    UNKNOWN
```

## Step 7: Create an ECS Service from a Compose File<a name="ECS_CLI_tutorial_compose_service"></a>

Now that you know that your containers work properly, you can make sure that they are replaced if they fail or stop\. You can do this by creating a service from your compose file with the ecs\-cli compose service up command\. This command creates a task definition from the latest compose file \(if it does not already exist\) and creates an ECS service with it, with a desired count of 1\.

Before starting your service, stop the containers from your compose file with the ecs\-cli compose down command so that you have an empty cluster to work with\.

```
ecs-cli compose down --cluster-config ec2-tutorial --ecs-profile tutorial-profile
```

Now you can create your service\.

```
ecs-cli compose service up --cluster-config ec2-tutorial --ecs-profile tutorial-profile
```

## Step 8: Clean Up<a name="ECS_CLI_tutorial_cleaning_up"></a>

When you are done with this tutorial, you should clean up your resources so they do not incur any more charges\. First, delete the service so that it stops the existing containers and does not try to run any more tasks\.

```
ecs-cli compose service rm --cluster-config ec2-tutorial --ecs-profile tutorial-profile
```

Now, take down your cluster, which cleans up the resources that you created earlier with ecs\-cli up\.

```
ecs-cli down --force --cluster-config ec2-tutorial --ecs-profile tutorial-profile
```