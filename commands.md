# 7777 Commands

### 7777

`7777` is an interactive command that will establish a tunnel with
the selected database. When running for the first time, it will
automatically deploy a CloudFormation Template with basic AWS
resources required to run the jump server on Fargate.

#### Options

##### --region

The AWS region running the RDS which you would like to connect.

`7777 --region eu-west-1`

##### --ttl

The maximum time the tunnel will be available (in hours). By default,
7777 runs a tunnel that automatically shuts down after 2 hours. This
helps avoid leaving containers running forever in case you forget
about them.

`7777 --ttl 24`

##### --forever

Alternatively, you may ignore `--ttl` and run your tunnel forever. 
It can be useful if you want to run a long migration process that
you're not aware how long it will take. `--forever` will allow you
to run a container that will only shutdown when you stop `7777`.

`7777 --forever`

##### --database

Removes the interactive step of `7777` and automatically selects
the RDS by its name.

`7777 --database my-rds-name-prod` 

##### --port

Customize the local port which 7777 uses to establish the SSH Tunnel.

`7777 --port 7780`  

##### --verbose

Prints informative messages about what 7777 is doing. Helpful for debugging.

`7777 --verbose`

--------------------

### 7777 list

List all currently running tasks on 7777 ECS Cluster. Always be
careful when shutting down your running containers in case it's currently
being used by a team member.

### 7777 stop

Stop **all** running containers.

##### --task

You may also specify just a single task to be stopped.

`7777 stop --task YOUR_TASK_ARN`

--------------------

### 7777 uninstall

Remove `7777` from your AWS account. This will delete the CloudFormation
stack created by 7777 and all of the resources created by 7777, such
as ECS Cluster, Task Definition, Security Group. 
**YOUR DATABASE WILL NEVER BE DELETED.**

`7777 uninstall`
