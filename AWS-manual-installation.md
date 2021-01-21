#### Manual installation - LIMITED SUPPORT

If you are not willing to provide an IAM role that allows
`iam:CreateRole` as well as `iam:PassRole`, you might
be able to still use 7777 if you wish to sacrifice the automatic
installation process.
Beware that support for this option is limited since it requires
advanced knowledge of AWS IAM and management of the template yourself.

The following permissions are still required for a day-to-day use.

```yaml
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Sid": "VisualEditor0",
            "Effect": "Allow",
            "Action": [
                "iam:PassRole",
                "rds:DescribeDBInstances",
                "rds:DescribeDBClusters",
                "ec2:DescribeSubnets",
                "ec2:DescribeRouteTables",
                "ec2:DescribeNetworkInterfaces",
                "ec2:describeSecurityGroups",
                "ec2:authorizeSecurityGroupIngress",
                "ecs:RunTask",
                "ecs:DescribeTasks",
                "ecs:ListTasks",
                "ecs:StopTask"
            ],
            "Resource": "*"
        }
    ]
}
```

As you may see, `iam:PassRole` is still necessary because
`aws ecs run-task` will need to receive the IAM Role necessary
to run the Fargate container.

##### Installation

The following CloudFormation template must be created:

```yaml
Description: Manual setup for 7777

Parameters:
  Vpc1:
    Type: AWS::EC2::VPC::Id

  RdsSecurityGroup1:
    Type: AWS::EC2::SecurityGroup::Id

Resources:
  7777Cluster:
    Type: AWS::ECS::Cluster
    Properties:
      ClusterName: 7777Cluster

  7777BastionTaskDefinition:
    Type: AWS::ECS::TaskDefinition
    Properties:
      ContainerDefinitions:
        - Essential: true
          Image: port7777/server:1
          LogConfiguration:
            LogDriver: awslogs
            Options:
              awslogs-group: !Ref 7777LogGroup
              awslogs-region: !Ref 'AWS::Region'
              awslogs-stream-prefix: task
          Name: 7777-bastion
          PortMappings:
            - ContainerPort: 22
          Privileged: 'false'
      Cpu: 256
      Memory: 512
      Family: 7777-bastion
      NetworkMode: awsvpc
      ExecutionRoleArn: !GetAtt 7777BastionTaskExecutionRole.Arn
      TaskRoleArn: !GetAtt 7777BastionTaskExecutionRole.Arn
      RequiresCompatibilities: [FARGATE]

  7777LogGroup:
    Type: AWS::Logs::LogGroup
    Properties:
      RetentionInDays: 7

  7777BastionTaskExecutionRole:
    Type: AWS::IAM::Role
    Properties:
      AssumeRolePolicyDocument:
        Statement:
          Effect: Allow
          Principal:
            Service: ecs-tasks.amazonaws.com
          Action: sts:AssumeRole
      ManagedPolicyArns:
        - arn:aws:iam::aws:policy/service-role/AmazonECSTaskExecutionRolePolicy

#####

  ContainerSecurityGroupForVpc1:
    Type: AWS::EC2::SecurityGroup
    Properties:
      GroupName: !Sub "7777-container-security-group-${Vpc1}"
      GroupDescription: '[7777] Security Group for Fargate container'
      VpcId: !Ref Vpc1
      SecurityGroupEgress:
        - IpProtocol: -1
          CidrIp: 0.0.0.0/0


  AuthorizeContainerSecurityGroupOnRds1:
    Type: AWS::EC2::SecurityGroupIngress
    Properties:
      Description: '[7777] Authorize Tunneling from Fargate Container'
      GroupId: !Ref RdsSecurityGroup1
      IpProtocol: -1
      SourceSecurityGroupId: !Ref ContainerSecurityGroupForVpc1
```

The `Vpc1` Parameter must be the Vpc in which the RDS is located
and the `RdsSecurityGroup1` must be a SecurityGroup attached to the
desired RDS.
You may add more `RdsSecurityGroup` parameters as well as more
`AuthorizeContainerSecurityGroupOnRds` resources to support multiple
databases. If you have databases in more than one Vpc, you'll also
need to add more `Vpc` as well as `ContainerSecurityGroupForVpc`.

##### Running 7777

When running 7777 with manual installation, you'll need to specify
the additional `--skip-install` parameter.

```shell
7777 --skip-install
```

This will skip the installation process and attempt to start 7777 on
an already installed infrastructure.

##### Disallow automatic Security Group authorization.

You may also opt to remove `ec2:authorizeSecurityGroupIngress` and
`ec2:describeSecurityGroups` fromthe IAM Policy documented above.
By doing so, you'll have to provide `--security-group` parameter with
the identifier of a Security Group that will be used for the
Fargate Container that allows communication with your RDS.
You may use the Security Group created by the template
documented above, but without automatic authorization if your user's
IP address is not allowed the tunnel will timeout. Alternatively,
you may allow ingress from `0.0.0.0/0` if you wish.
