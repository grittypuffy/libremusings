+++
page_template = "blog.html"
template = "post.html"
title = "Self-hosting Ente for preserving those special moments"
description = "No-gyan approach I used to self-host Ente"
date = "2025-04-23"
insert_anchor_links = "right"
generate_feeds = true
lang = "en"
tags = ["foss", "privacy", "selfhosting"]
[extra]
title = "Self-hosting Ente for preserving those special moments"
date_format = "%Y-%m-%d"
lang = "en"
categorized = false # posts can be categorized
back_to_top = true # show back-to-top button
toc = true # show table-of-contents
comment = false # enable comment
copy = true # show copy button in code block
outdate_alert = false
outdate_alert_days = false
reaction = false
outdate_alert_text_before = false
+++

# Why Ente?

I like it's technical architecture. Yes, I am aware of [Immich](https://immich.app) (which I believe is a cool app with great documentation and impressive backstory), but the architecture and modularity of Museum enough to be used in other services like Ente Auth makes it more appealing to my nerdy side.

I am not going to discuss about the privacy features of Ente, or why you should self-host Ente and not use Google Photos or other proprietary applications in this post, saving that for later.

# Requirements

Ente's server (museum) is a lightweight Go binary, thus it requires minimal resources. Thus the following are needed for self-hosting:
- A server with minimal requirements (I used `t2.small` instance on AWS with Debian AMI)
- A domain. You need one for your main photos service, whereas you are free to use separate subdomains for the museum (backend of Ente), album (best approach as it provides separation), and storage (you will have to enable CORS)
- `nginx`, `caddy` or `traefik` for the web server

# Getting started

## Installing dependencies

Ente's official documentation recommends usage of Docker Compose for getting started with Ente by using [quickstart.sh](https://help.ente.io/self-hosting/)

Installing Docker, Docker Compose and cURL is needed to get started with it.

### Docker and Docker Compose

Setting up the repository and installing the needed package from [this guide](https://docs.docker.com/engine/install/debian/#install-using-the-repository) is needed for installing the latest version of Compose, as `quickstart.sh` needs Compose version to be greater than 2.30

Trying to run the script as a non-root user would require some additional steps, described in the [official documentation](https://docs.docker.com/engine/install/linux-postinstall/)

## Setting up other dependencies

Install them by the following command:

``` sh
sudo apt install curl caddy
```

Installation of Caddy should automatically generate a `Caddyfile` used for configuration of services at `/etc/caddy/Caddyfile`.

## Running `quickstart.sh`


``` sh
sh -c "$(curl -fsSL https://raw.githubusercontent.com/ente-io/ente/main/server/quickstart.sh)"
```

This should retrieve the script, create a folder named `my-ente` on your working directory and start the containers for the following services:


|Service                                  |Port   |
|-----------------------------------------|-------|
| Web application                         | 3000  |
| Museum (backend)                        | 8080  |
| Minio (S3-compatible storage)           | 3200  |
| PostgreSQL (database)                   | 5432  |
| socat (Helper. Networking in containers)|       |

## Configuring the web server using Caddy

Caddy is an open-source web server which generates SSL/TLS certificates automatically for the domains from Let's Encrypt, making it more easier to set up than using nginx where you're responsible for generation and renewal of certificates.

### Domain configuration

Create an A (or AAAA) record on your subdomain (if you're hosting multiple services on different subdomains) or on your domain and enter the IP address of the server. Please note that DNS propagation should take some time.

### Edit `Caddyfile`

Once the containers are up and running, edit your `Caddyfile` to use your prefered subdomain that's been configured.
    
Here's the configuration that I used:

```
<photos-domain> {
    handle_path /api/* {
        reverse_proxy localhost:8080 {
            header_up Host photos.example.com
        }
    }
    handle {
        reverse_proxy localhost:3000
    }
    header {
        Host {host}
        X-Forwarded-For {remote_host}
        X-Forwarded-Proto {scheme}
        Access-Control-Allow-Methods "GET, POST, PUT, DELETE"
        Access-Control-Allow-Headers "*"
        Access-Control-Max-Age "3000"
        Access-Control-Expose-Headers "Etag"
    }
}

<album-domain> {
    reverse_proxy localhost:3002
}

<minio-domain> {
    handle {
        reverse_proxy localhost:3200
    }
}
```

I have prefixed the backend service with `/api` path in the web service to avoid dealing with another subdomain.

The host header is needed for validation of pre-signed signature.

I have used separate subdomains for the album and the MinIO storage service.

## Modify the generated configuration

1. In `museum.yaml`, modify the following properties to the values below if you're going to use a subdomain for storage:

    ``` yaml
    s3:
        are_local_buckets: false
        use_path_style_urls: true
        b2-eu-cen:
            endpoint: https://yoursubdomain.com
    ```
2. In `compose.yaml`, uncomment the ports for the services that you're going to use, by default, ports `3000` and `3002` are open for the photos web application and album respectively.
3. In `compose.yaml`, edit the `ENTE_API_ORIGIN` and `ENTE_ALBUMS_ORIGIN` with your endpoints.

## Configure CORS for MinIO

While using a subdomain for MinIO, it is required to configure CORS which can be done by using `mc` (CLI client for MinIO)

Ensure the service is running and assuming your MinIO container is named `my-ente-minio-1`, set an alias for your storage bucket. Do this for the bucket named `b2-eu-cen` since that is the one used as a primary (hot) storage.

``` sh
docker exec -it my-ente-minio-1 sh -c 'mc alias set <storage-alias-name> <minio-endpoint> <minio-key> <minio-secret>
```

Now allow specific origins or all using wildcard (`*`) by this command:

``` sh
docker exec -it my-ente-minio-1 sh -c 'mc cors set ente-storage/b2-eu-cen api cors_allow_origin="*"'
```

## Test the setup

Once everything has been done, test your setup by reloading or restarting Caddy.

``` sh
sudo systemctl reload caddy
sudo systemctl restart caddy
```

You should be able to access Ente in your configured domain. Create a new account, the first account is considered as the admin user who can manage the platform.

It should ask for a verification code which is present in compose logs that can be accessed by

``` sh
sudo docker compose logs
```

You should be able to upload photos and create albums. Happy preserving!

# Post-setup

There are some things that can be done to safeguard your installation and gain maximum value out of your self-hosted instance.

## Increasing the storage quota

This is to be done by using Ente CLI. Configure your endpoint using the `config.yaml` after downloading the binary from their releases using cURL.

``` yaml
endpoint:
    api: http://localhost:8080
```

Assuming you've aliased the binary, add an account. It should prompt for your credentials. Set the environment variable `ENTE_CLI_SECRETS_PATH` to `./secrets.txt` (assuming the file is in same directory as the binary, or use `ENTE_CLI_CONFIG_DIR` for setting the directory)

``` sh
ente account add
```

Update your subscription by adding the admin's ID by checking for the ID using the following command:

``` sh
ente account list # Get ID. Add this to internal.admin in museum.yaml
ente admin update-subscription -u grittypuffy@riseup.net --no-limit True -a grittypuffy@riseup.net
```

## Disabling new registrations

In `museum.yaml`, add this configuration:

``` yaml
internal:
    disable-registration: true
```

## Migrate to Backblaze or other S3-compatible storage service

If you want more reliability and backups, enable cold storage via the configuration file and use other external services such as Backblaze or Wasabi or AWS S3 for storing your files.
