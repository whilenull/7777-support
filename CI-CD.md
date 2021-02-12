# Using 7777 in CI/CD

It is possible to use 7777 in CI/CD.

For example, some applications need to connect to the production database to generate caches before deploying.
Some other applications need to run database migrations before deploying.

*Note: running 7777 in CI requires a "Team" license because 7777 will run on a different machine every time.*

## Run 7777 using Docker

Running 7777 via Docker is simpler as it avoids downloading the `7777` binary on every run.

Here is an example:

```bash
docker run --rm -e AWS_ACCESS_KEY_ID -e AWS_SECRET_ACCESS_KEY -e PORT_7777_LICENSE -p 7777:7777 port7777/7777:1 --verbose
```

Define `AWS_ACCESS_KEY_ID`, `AWS_SECRET_ACCESS_KEY` and `PORT_7777_LICENSE` as secret environment variables in your CI system. These variables will be forwarded to 7777 in the container, and will be used to configure the AWS credentials as well as the 7777 license key.

Since the execution will not be interactive, it is also recommended to explicitly specific the region and database name:

```bash
docker run --rm -e AWS_ACCESS_KEY_ID -e AWS_SECRET_ACCESS_KEY -e PORT_7777_LICENSE -p 7777:7777 port7777/7777:1 --region=us-east-1 --database=my-database-name --ttl=1 --verbose
```

The `--ttl=1` let us shorten the tunnel timeout to 1h: it is unlikely our build will run for longer.

## Wait for the connection to be established

One common need in CI is to wait for the tunnel to be opened before executing the next commands. Here is an example:

```bash
docker run --rm -e AWS_ACCESS_KEY_ID -e AWS_SECRET_ACCESS_KEY -e PORT_7777_LICENSE -p 7777:7777 port7777/7777:1 --region=$AWS_REGION --database=$DATABASE_NAME --ttl=1 --verbose &
# Wait until the tunnel is ready
while ! nc -z localhost 7777; do sleep 1; done;
# Now the tunnel is open, our code can connect to `localhost:7777`
# ...
```

As you can see above, we run `7777` in background using `&`, waiting until the port is open using `netcat` in a loop.

## GitLab CI

On GitLab, Docker containers run inside Docker containers (aka "docker in docker").

This messes up the domain names (we can simply connect to `localhost`).

Here is a complete working example:

```yaml
variables:
  AWS_REGION: us-east-1
  DATABASE_NAME: my-rds-db-name
  DB_HOST: 'docker:7777'
build:
  image: docker:19.03.12
  services:
    - docker:19.03.12-dind
  script:
    # Open a tunnel to the RDS database in background using 7777
    # `--name database` will make the tunnel accessible under the domain name "database"
    - docker run --rm --name database -e AWS_ACCESS_KEY_ID -e AWS_SECRET_ACCESS_KEY -e PORT_7777_LICENSE -p 7777:7777 port7777/7777:1 --region=$AWS_REGION --database=$RDS_DATABASE_NAME --ttl=1 --verbose &
    # Wait until the tunnel is ready
    # Here we use the domain name "docker" because that's the name of the Docker service (see the "services" key above)
    - while ! nc -z docker 7777; do sleep 1; done;
    # Now we can connect to the tunnel
    - mysql -h docker -P 7777 ... # at the root, use "docker"
    - docker run --rm --link database:database mysql mysql -h database -P 7777 ... # inside another container, use "database"
```

Long story short:

- to connect to the tunnel from the build container: use `docker:7777`
- to connect to the tunnel from another container running inside the build container: link to the database container (`--link database:database`) and use `database:7777`
