Welcome to the documentation of [7777](https://port7777.com/).

This repository is also the place to ask for support about 7777: [**open a new GitHub issue**](https://github.com/whilenull/7777-support/issues/new) and detail your problem.

If you need to discuss a problem with a purchase or a license, contact us privately (our email address can be found in the footer of [port7777.com](https://port7777.com/)).

## Installation and update

7777 is bundled as a self-contained binary.

### Linux

Download and install the Linux binary in `/usr/local/bin`:

```bash
wget -q https://releases.port7777.com/latest/linux/7777
chmod +x 7777
sudo mv 7777 /usr/local/bin
```

*Note: we've had one report that Bitdefender on Ubuntu was deleting the `7777` executable. If that happens to you, please let us know. You can add 7777 to Bitdefender's exclusion list.*

A Linux Alpine version is available under the https://releases.port7777.com/latest/alpine/7777 URL.

*Note: Due to [a limitation in `pkg`](https://github.com/vercel/pkg/issues/726), running `apk add libstdc++ libgcc` is required with Linux Alpine.*

### MacOS

Download and install the MacOS binary in `/usr/local/bin`:

```bash
curl -o 7777 https://releases.port7777.com/latest/macos/7777
chmod +x 7777
sudo mv 7777 /usr/local/bin
```

7777 is also available via Brew: `brew install 7777`

### Windows

Download the Windows 10 binary directly from https://releases.port7777.com/latest/windows/7777.exe or via the command line:

```bash
curl.exe -o 7777.exe https://releases.port7777.com/latest/windows/7777.exe
```

Then run the `7777.exe` program, for example using [Git Bash](https://gitforwindows.org/).

## License

7777 requires a valid license to run. To set it up, run:

```bash
7777 configure YOUR_LICENSE
```

Replace `YOUR_LICENSE` with the license sent to you after your purchase on [port7777.com](https://port7777.com/) (you will receive it by email). This command will store your license in a `~/.7777/license` file on your machine.

If you prefer, you can also set your license in a `PORT_7777_LICENSE` environment variable.

## AWS access keys

To connect to a database in your AWS account, 7777 requires AWS permissions.

By default, 7777 behaves like [the official `aws` CLI](https://aws.amazon.com/cli/) and picks up any AWS account configured on your machine in the  `~/.aws/credentials` file. That means that if you use the AWS CLI, 7777 should work out of the box.

If you want to use a specific AWS profile, set the `AWS_PROFILE` variable. For example:

```bash
AWS_PROFILE=my-profile 7777
```

If you prefer, you also can provide AWS access keys explicitly via the following environment variables:

- `AWS_ACCESS_KEY_ID`
- `AWS_SECRET_ACCESS_KEY`

You can find [the list of IAM permissions here](AWS-permissions.md) if you create access keys specific to 7777.

## Usage

To use 7777 to connect to an AWS RDS database, simply run `7777`:

```bash
7777
# Use the --verbose flag to show more details
7777 --verbose
# Use the --region option to force a specific region
7777 --region=us-east-1
```

The first time, 7777 will setup the strict minimum it requires to run in your AWS account (the list is detailed below).

7777 will then look at all the available databases and let you select which one to connect to.

Then, 7777 will establish a secure connection (also called "tunnel") between your computer and the AWS database. As long as 7777 is running, the connection will remain active.

By default, the database will be accessible through port `7777` on your computer.

#### Accessing the database

The database can be accessed through `localhost` via port 7777, using the credentials you use in production.

Some database clients offer the option to connect via an SSH connection: this is not needed as 7777 already took care of that. Just point your client to `127.0.0.1:7777`.

For example, using the CLI MySQL client:

```bash
mysql --user=admin --password=XXX --host=127.0.0.1 --port=7777
```

### Options

A list of all commands and options is [available here](https://github.com/whilenull/7777-support/blob/main/commands.md).

#### Region

The AWS region can be customized via the `7777 --region=us-east-1` option.

## Elasticache support

Starting on 1.1.0, 7777 will now support Elasticache for Redis.
In order to establish a connection to your Redis Cluster, a new
parameter `--elasticache` has been added. Make sure you have (permission to list Elasticache instances)[https://github.com/whilenull/7777-support/blob/main/AWS-permissions.md#upgrading-from-10].

After the tunnel is established you may connect to your Redis cluster using any tool you like on your local port 7777

```bash
redis-cli -p 7777
```

## Running in Docker and CI/CD

It is possible to run 7777 in Docker, as well as use it in CI/CD.

See the [**CI/CD documentation**](CI-CD.md).

## How it works

7777 lets you connect to a remote database running in the cloud via a port on your machine. This is possible through something called **an SSH tunnel**.

SSH is a secure protocol and _SSH tunnels_ are a way to use that protocol to forward ports from one machine to another one.

Here, we forward port 7777 of your computer to the port of your database running in the cloud.

7777's job is then to setup everything needed, and then start the SSH tunnel. Here it is in more details:

- Start a jump server (also known as "bastion") in the VPC.

  This server is used for the SSH tunnel to allow your computer to access the RDS database. Indeed, that database is not publicly accessible.

  The jump server is actually a Fargate container, which is more lightweight than an EC2 instance. It is also much cheaper since we only pay when the container is running (i.e. when 7777 runs).

- Start an SSH server (`sshd` daemon) in the jump server.

- Generate a unique private and public SSH key and set it up on the SSH server.

  By using random and autogenerated SSH keys, we don't need one for you and we keep things secure by not reusing the same key multiple times.

- Start the SSH tunnel from port 7777 on your machine to the database, via the jump server.

When you stop 7777 via `Ctrl+C`, the SSH tunnel and the Fargate container are stopped and destroyed.

If you connect to your database for 10 minutes, then you would only pay for 10 minutes of Fargate, which is $0.002 (0.2 cents).

### AWS stack details

In order to provide a cheap, secure, and fast tunnel to databases, 7777 uses serverless Fargate containers as "jump servers".

**You don't have to understand or even care about all of this**, but in case you want to know, here is what 7777 sets up on the first run:

- to run the bastion container: an ECS Fargate cluster, a "task definition" and the associated IAM role (cost: free)
- to store logs from the SSH server on the container: a log group, logs are purged after 7 days (cost: free)
- a security group for the container, and a rule allowing that security group to access the database

Then, each time 7777 runs and the connection is active, the following AWS resources run:

- Fargate container (cost: ~$0.01/hour)

You can easily review all those resources as they are grouped in a single CloudFormation stack: [search for `port7777` in the AWS CloudFormation console](https://console.aws.amazon.com/cloudformation/home#/stacks?filteringText=port7777&filteringStatus=active&viewNested=true&hideStacks=false).

## Uninstall

To remove everything that 7777 has set up in your AWS account, run:

```bash
7777 uninstall
```

You can also delete everything manually by deleting the `port7777` CloudFormation stack in the AWS console. To find it, [open the CloudFormation console and search for port7777](https://console.aws.amazon.com/cloudformation/home#/stacks?filteringText=port7777&filteringStatus=active&viewNested=true&hideStacks=false) (make sure to select the correct region), then delete it.

Note: your databases will _not_ be deleted. Only what 7777 used to create the tunnel is deleted. Just to be clear one last time: your database is safe.

## Troubleshooting

> Conflicts with other tools deleting changes made by 7777.

Some tools, like Terraform, can sometimes overwrite changes made by 7777. For example, they could remove from your database the security group that 7777 adds.
In case that happens, this is very hard to detect and fix these issues because 7777 uses CloudFormation, which itself does not fix changes made externally to the CloudFormation stack.

One workaround is to run `7777 uninstall` and `7777` again.

> Error: There is no subnet routing to the internet in availability zone us-east-1a of VPC vpc-xxxxx.

This error can happen for applications that use the default VPC provided in the region. Those default VPCs are unfortunately not explicitely associated with the Route Table.

Not sure what that means or what to do? Please have a look [at this answer](https://github.com/whilenull/7777-support/issues/5#issuecomment-733593007), it should help. Get in touch if you have any issue.
