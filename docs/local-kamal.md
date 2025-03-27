# Introduction

This guide will walkthrough the setup of deploying a Rails application completely locally.

> [!IMPORTANT]
> This guide assumes you're on macOS whether it's Intel or Apple Silicon.

# Components

## Docker

Kamal uses Docker to build the image on the host machine. As a prerequisite, Docker needs to be installed natively on your machine.

> [!NOTE]
> There's a subcomponent within Docker that's created by Kamal which a builder container. We'll see it later in the guide.

## Multipass

Multipass is a tool developed by Canonical (Ubuntu) to easily launch Ubuntu VMs to simulate the Ubuntu Server environment.

Since we don't want to test this on a sever (physical or cloud), this will help to launch a VM on your machine and SSH into it.

## Local Docker Registry

Kamal creates a Docker image and by default expects to provide credentials to DockerHub which is a cloud based Docker registry.

Because our objective is to have everything running locally, we'll spin up our own registry locally using the `registry` image.

## Rails Application

A Rails 8 application that has a Dockerfile created. This will also include the default `deploy.yaml` file for Kamal.

## PostgreSQL

> [!WARNING]
> Under Construction 🚧

# Setup

Let's walk through each component and make sure we have all we need to start deploying using Kamal locally.

## Private IP Address

We need to grab the private IP address of your machine. We can't use `localhost` or `127.0.0.1` because we'll be connecting to our host machine from multiple VMs/containers.

This can be achieved by running in the terminal:

```bash
ifconfig 
```

> [!WARNING]
> The IP address depends on the network you're connected to. If you switch from the office to home, for instance, the IP address would change and it needs to be reflected to wherever it's used.

## Local Registry

### Setup
To instantiate a local registry, we'll start a container by running the following:

```bash
docker run -d -p 5000:5000 --restart always --name registry registry:2

```
To test if it's working correctly, run the below, and you should see responses coming in:

```bash
ping <YOUR_PRIVATE_IP>
```

### Login

To create the user and password to login to the registry, run the following. It will prompt you to enter the username followed by the password.
For this guide, we'll use **registry** for both the username and password.

```bash
docker login <YOUR_PRIVATE_IP>:5000
```

## Multipass

### Installation

Using homebrew, install multipass:

```bash
brew install multipass
```

### Launch VM

Run the following to create the new VM using multipass:

```bash
multipass launch --name kamal-vm --memory 4G --disk 10G --bridged
```

It will take some time to boot the VM, and once it's done you can test the access by running:

```bash
multipass shell kamal-vm
```

Also to get the IP address of the VM, which is needed later to configure Kamal, run the following:

```bash
multipass info kamal-vm
```

### SSH Setup

Kamal uses `ssh` to access the server (VM in our case). Before we can login with `ssh`, we'll need to create a private/public key pair.

On your machine, run the following to create a key pair:

```bash
ssh-keygen -t ed25519 -C "testing-kamal@example.com" -f ~/.ssh/kamal_vm
```

> [!TIP]
> You can skip setting passphrases by pressing *Enter* twice. Since we're testing it locally, security isn't a concern.

Next, we'll need to add the key to the ssh agent by running:

```bash
ssh-add ~/.ssh/kamal_vm
```

To confirm if the key is added, run the following to list all identities registered to the agent. You might see more than one.

```bash
ssh-add -L
```

Finally, we push the public to the VM using the following command:

```bash
multipass exec kamal-vm -- sh -c "mkdir -p /home/ubuntu/.ssh && echo '$(ssh-add -L | awk 'NR==1')' >> /home/ubuntu/.ssh/authorized_keys"
```

> [!NOTE]
> Note the `NR=1`, replace `1` with the number where the identity appeared when running `ssh-add -L`. For example, if the *kamal_vm* identity was the **third** item, then it's `NR=3`.


Once the above is done, test ssh-ing to the VM by running which will lead you log you in to the shell.

```bash
ssh ubuntu@<VM_IP_ADDRESS>
```

### Bootstrap the VM

Before Kamal can deploy to the VM, some packages need to be installed, specifically Docker. Run the following to install the required packages.

```bash
multipass exec kamal-vm -- sh -c "sudo apt update && sudo apt upgrade -y && sudo apt install -y docker.io curl git && sudo usermod -a -G docker ubuntu"
```



## Insecure Shenanigans TODO:

Docker by default doesn't allow to connect to registries that are not secure, aka *https*.
Since we're in a local machine and it's for testing purposes, we can go around it by configure Docker to whitelist our IP and allow pulling/pushing to the local registry.

There are three touch points to where we'll apply this:

### Docker on the Host

On your machine, open the Docker Desktop application and navigate to Settings -> Docker Engine.

In the text box, add to the JSON the following:

```json
  "insecure-registries": [
    "<YOUR_PRIVATE_IP>:5000"
  ]
```


### VM

Since the VM will pull from the registry, it too needs to whitelist our IP.

- SSH into the VM by running `ssh ubuntu@<VM_IP_ADDRESS>`
- Create a docker engine config file by running `sudo touch /etc/docker/daemon.json`
- Using `nano` or `vi`, add the JSON
```
{
  "insecure-registries": [
    "<YOUR_PRIVATE_IP>:5000"
  ]
}
```

> [!NOTE]
> Make sure to run the editor using sudo, since it's a protected directory.

### Docker Builder

Come back here once Kamal config is complete.


Ok, so you've reached the last step and it errors out. That is expected, because Kamal uses a special kind of docker containers called builder containers.
Since it's another container, we need to let the builder know that it's ok to pull from an insecure registry.

This container is created by Kamal, that's why we need it to fail once before we can access the builder container.

To get the name of the container, run `docker ps` and look for the container with the name containing `kamal-local-docker`. For example:

```
➜ docker ps
CONTAINER ID   IMAGE                           COMMAND                  CREATED        STATUS         PORTS                    NAMES
3d5f0087e54a   moby/buildkit:buildx-stable-1   "buildkitd --allow-i…"   20 hours ago   Up 9 minutes                            buildx_buildkit_kamal-local-docker-container0 <===========
43999a204647   registry:2                      "/entrypoint.sh /etc…"   21 hours ago   Up 9 minutes   0.0.0.0:5000->5000/tcp   registry
```

Now we need to access the shell of this container and to do that we run 

```bash
docker exec -it <CONTAINER_NAME || CONTAINER_ID> sh
```

Once we're in the shell, create a directory under `/etc` called `buildkit`.

```bash
mkdir /etc/buildkit
```

Then create a file called `buildkitd.toml` and add the contents below to the file.

```toml
[registry."<YOUR_PRIVATE_IP>:5000"]
  http = true
```

Exit the shell and restart the container so that the configuration takes effect.

```bash
docker restart <CONTAINER_NAME || CONTAINER_ID>
```

## Kamal Config

After we've setup the prerequisites, we'll not fill in the config files in the Rails application.
We'll be concerned mainly with two files in the Rails application.

- config/deploy.yaml
- .kamal/secrets

### .kamal/secrets

Let's start with the easy one. Replace `$KAMAL_REGISTRY_PASSWORD` with the password `registry`.

### config/deploy.yaml

- Set the `image` property to `registry/store`.
- Under `servers->web` replace the IP address with the IP of the VM.
- Comment out the `proxy` section
- Update the `username` under `registry` property to be `registry`
- Add `server` property besides `username` under `registry` with your private IP (`<YOUR_PRIVATE_IP>:5000`)
- Uncomment the `ssh` property and set the `user` to `ubuntu`

Look for the arrows below to see where the changes are expected


```yaml
# Name of your application. Used to uniquely configure containers.
service: store

# Name of the container image.
image: registry/store # <==============================

# Deploy to these servers.
servers:
  web:
    - <VM_IP_ADDRESS> # <==============================
  # job:
  #   hosts:
  #     - 192.168.0.1
  #   cmd: bin/jobs

# Enable SSL auto certification via Let's Encrypt and allow for multiple apps on a single web server.
# Remove this section when using multiple web servers and ensure you terminate SSL at your load balancer.
#
# Note: If using Cloudflare, set encryption mode in SSL/TLS setting to "Full" to enable CF-to-app encryption.
#proxy: # <==============================
#  ssl: true # <==============================
#  host: app.example.com # <==============================

# Credentials for your image host.
registry:
  # Specify the registry server, if you're not using Docker Hub
  # server: registry.digitalocean.com / ghcr.io / ...
  username: registry # <==============================
  server: <YOUR_PRIVATE_IP>:5000 # <==============================

  # Always use an access token rather than real password when possible.
  password:
    - KAMAL_REGISTRY_PASSWORD

# Inject ENV variables into containers (secrets come from .kamal/secrets).
env:
  secret:
    - RAILS_MASTER_KEY
  #clear:
    # Run the Solid Queue Supervisor inside the web server's Puma process to do jobs.
    # When you start using multiple servers, you should split out job processing to a dedicated machine.
    # SOLID_QUEUE_IN_PUMA: true

    # Set number of processes dedicated to Solid Queue (default: 1)
    # JOB_CONCURRENCY: 3

    # Set number of cores available to the application on each server (default: 1).
    # WEB_CONCURRENCY: 2

    # Match this to any external database server to configure Active Record correctly
    # Use store-db for a db accessory server on same machine via local kamal docker network.
    # DB_HOST: 192.168.0.2

    # Log everything from Rails
    # RAILS_LOG_LEVEL: debug

# Aliases are triggered with "bin/kamal <alias>". You can overwrite arguments on invocation:
# "bin/kamal logs -r job" will tail logs from the first server in the job section.
aliases:
  console: app exec --interactive --reuse "bin/rails console"
  shell: app exec --interactive --reuse "bash"
  logs: app logs -f
  dbc: app exec --interactive --reuse "bin/rails dbconsole"


# Use a persistent storage volume for sqlite database files and local Active Storage files.
# Recommended to change this to a mounted volume path that is backed up off server.
volumes:
  - "store_storage:/rails/storage"


# Bridge fingerprinted assets, like JS and CSS, between versions to avoid
# hitting 404 on in-flight requests. Combines all files from new and old
# version inside the asset_path.
asset_path: /rails/public/assets

# Configure the image builder.
builder:
  arch: arm64

  # # Build image via remote server (useful for faster amd64 builds on arm64 computers)
  # remote: ssh://docker@docker-builder-server
  #
  # # Pass arguments and secrets to the Docker build process
  # args:
  #   RUBY_VERSION: ruby-3.4.2
  # secrets:
  #   - GITHUB_TOKEN
  #   - RAILS_MASTER_KEY

# Use a different ssh user than root
ssh: # <==============================
  user: ubuntu # <==============================

# Use accessory services (secrets come from .kamal/secrets).
# accessories:
#   db:
#     image: mysql:8.0
#     host: 192.168.0.2
#     # Change to 3306 to expose port to the world instead of just local network.
#     port: "127.0.0.1:3306:3306"
#     env:
#       clear:
#         MYSQL_ROOT_HOST: '%'
#       secret:
#         - MYSQL_ROOT_PASSWORD
#     files:
#       - config/mysql/production.cnf:/etc/mysql/my.cnf
#       - db/production.sql:/docker-entrypoint-initdb.d/setup.sql
#     directories:
#       - data:/var/lib/mysql
#   redis:
#     image: redis:7.0
#     host: 192.168.0.2
#     port: 6379
#     directories:
#       - data:/data
```

## Deploying with Kamal

To deploy the application, we run the kamal cli tool in the root of the Rails project:

```bash
./bin/kamal deploy
```

This will fail with an error like:

```
ERROR: failed to solve: failed to push 172.17.157.199:5000
```

This is expected, and this would be last hoop to jump. Refer to [Docker Builder](#Docker-Builder)

Once the error is resolved, the deployment should be successful. To test the instance running, visit `http://<VM_IP_ADDRESS>`
