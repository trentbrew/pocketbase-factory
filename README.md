# 1. Create the PocketBase Dockerfile:

```
FROM alpine:latest

ARG PB_VERSION=0.10.1

RUN apk add --no-cache \
    unzip \
    # this is needed only if you want to use scp to copy later your pb_data locally
    openssh

# download and unzip PocketBase
ADD https://github.com/pocketbase/pocketbase/releases/download/v${PB_VERSION}/pocketbase_${PB_VERSION}_linux_amd64.zip /tmp/pb.zip
RUN unzip /tmp/pb.zip -d /pb/

EXPOSE 8080

# start PocketBase
CMD ["/pb/pocketbase", "serve", "--http=0.0.0.0:8080"]

```

# 2. Install Flyctl

Follow the installation instructions from https://fly.io/docs/hands-on/install-flyctl/.

Run `flyctl auth signup` to create a Fly.io account (email or GitHub).

Run `flyctl auth login` to login.

# 3. Launch (aka. machine setup)

Navigate to the directory where the Dockerfile from 1) was created.

Run `flyctl launch` (optionally you can specify --dockerfile if you are running from different directory).

This will prompt you a couple questions like the name of your app, deployment region, etc.

Type n for the last 2 questions asking for creating a Postgresql database and whether you want to deploy directly.

```
? Would you like to set up a Postgresql database now? No

? Would you like to deploy now? No
```

The launch command will create a fly.toml configuration file in the working directory.

# 4. Create and mount 1GB free persistent volume

Run `flyctl volumes create pb_data --size=1`

It will prompt you to select a region - preferably choose the same region as in 3).

Once finished, open `fly.toml` and add somewhere at the root level the following `mounts` config:

```
[mounts]
  destination = "/pb/pb_data"
  source = "pb_data"
```

Your `fly.toml` should look like this:

```
# fly.toml file generated for pocketbase-test on 2022-09-18T12:44:18+03:00

app = "pocketbase-test"
kill_signal = "SIGINT"
kill_timeout = 5
processes = []

[experimental]
  allowed_public_ports = []
  auto_rollback = true

[mounts]
  destination = "/pb/pb_data"
  source = "pb_data"

# optional if you want to change the PocketBase version
[build.args]
  PB_VERSION="0.11.0"

[[services]]
  http_checks   = []
  internal_port = 8080
  processes     = ["app"]
  protocol      = "tcp"
  script_checks = []
  [services.concurrency]
    hard_limit = 25
    soft_limit = 20
    type       = "connections"

  [[services.ports]]
    force_https = true
    handlers    = ["http"]
    port        = 80

  [[services.ports]]
    handlers = ["tls", "http"]
    port     = 443

  [[services.tcp_checks]]
    grace_period  = "1s"
    interval      = "15s"
    restart_limit = 0
    timeout       = "2s"
```

# 5. Deploy

Run `flyctl deploy` to deploy the app.

To access your PocketBase dashboard navigate to `https://<your-app-name>.fly.dev/_/`.

To deploy new configuration/image changes, just run `flyctl deploy` again.

## Backup and downloading a copy of `pb_data`

Fly.io will even create a daily snapshot automatically for your volume (it will be kept for 5 days).

To download a copy of your remote `pb_data` directory to your local machine, you could tun the following commands in 2 separate terminals:

```
# this will register a ssh key with your local agent (if you haven't already)
flyctl ssh issue --agent

# proxies connections to a fly VM through a Wireguard tunnel
flyctl proxy 10022:22

# run in a separate terminal to copy the pb_data directory
scp -r -P 10022 root@localhost:/pb/pb_data  /your/local/pb_data
```
