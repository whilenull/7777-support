Here are the IAM permissions that 7777 requires to run it correctly:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "Install",
      "Effect": "Allow",
      "Action": [
        "iam:GetRole",
        "iam:PassRole",
        "iam:CreateRole",
        "iam:AttachRolePolicy",
        "iam:CreateServiceLinkedRole",
        "logs:CreateLogGroup",
        "logs:PutRetentionPolicy",
        "logs:TagResource",
        "ec2:CreateSecurityGroup",
        "ec2:AuthorizeSecurityGroupEgress",
        "ec2:createTags",
        "ecs:DescribeClusters",
        "ecs:CreateCluster",
        "ecs:RegisterTaskDefinition",
        "ecs:DeregisterTaskDefinition",
        "cloudformation:GetTemplate",
        "cloudformation:CreateChangeSet",
        "cloudformation:DescribeChangeSet",
        "cloudformation:ExecuteChangeSet",
        "cloudformation:DescribeStacks"
      ],
      "Resource": "*"
    },
    {
      "Sid": "Uninstall",
      "Effect": "Allow",
      "Action": [
        "iam:DeleteRole",
        "iam:DetachRolePolicy",
        "logs:DeleteLogGroup",
        "ec2:DeleteSecurityGroup",
        "ec2:RevokeSecurityGroupEgress",
        "ec2:RevokeSecurityGroupIngress",
        "ecs:DeleteCluster",
        "cloudformation:DeleteStack"
      ],
      "Resource": "*"
    },
    {
      "Sid": "Firewall",
      "Effect": "Allow",
      "Action": [
        "ec2:AuthorizeSecurityGroupIngress"
      ],
      "Resource": "*"
    },
    {
      "Sid": "Tunnel",
      "Effect": "Allow",
      "Action": [
        "ec2:DescribeSubnets",
        "ec2:DescribeRouteTables",
        "ec2:DescribeSecurityGroups",
        "ec2:DescribeNetworkInterfaces",
        "ecs:RunTask",
        "ecs:DescribeTasks",
        "ecs:ListTasks",
        "ecs:StopTask"
      ],
      "Resource": "*"
    },
    {
      "Sid": "RDS",
      "Effect": "Allow",
      "Action": [
        "rds:DescribeDBInstances",
        "rds:DescribeDBClusters",
        "rds:DescribeDBSubnetGroups"
      ],
      "Resource": "*"
    },
    {
      "Sid": "Elasticache",
      "Effect": "Allow",
      "Action": [
        "elasticache:DescribeCacheClusters"
      ],
      "Resource": [
        "*"
      ]
    }
  ]
}
```

### Upgrading from 1.0

If you're upgrading from 1.0 to 1.1 and want to use Elasticache for Redis, make sure to
add the following permission to your existing access:

```json
    {
        "Sid": "Elasticache",
        "Effect": "Allow",
        "Action": [
            "elasticache:DescribeCacheClusters"
        ],
        "Resource": [
            "*"
        ]
    }
```

For a more restrictive permission setup, check out the  
[Manual Installation process (Support Limited)](https://github.com/whilenull/7777-support/blob/main/AWS-manual-installation.md)
