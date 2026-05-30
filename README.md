
# Table of Contents

1.  [Introduction](#org767d075)
2.  [Setup](#orgbc55725)
3.  [Prerequisites](#orgeb7cfce)
    1.  [Docker](#orga4cc88b)
    2.  [Tailscale](#org346097c)
4.  [Services](#orgdc818e0)
    1.  [Portainer](#org4d7e6ee)
    2.  [Nextcloud](#orga3f5683)
    3.  [Nextcloud Whiteboard](#org685d0c0)
    4.  [Bitwarden](#orgafa548e)
    5.  [FreshRSS](#orgb126edf)
5.  [Backups](#org323cf64)



<a id="org767d075"></a>

# Introduction

In this repository I have listed all of the steps I have taken to configure my Raspberry Pi 5 for a homelab. This serves as documentation that I can get back to if I want to look something up, and hopefully it will be helpful to others too.


<a id="orgbc55725"></a>

# Setup

-   **Device:** Raspberry Pi 5 Model B
-   **Memory:** 8GB RAM
-   **OS:** Debian GNU/Linux 13 (trixie) aarch64 &mdash; *This is Pi OS Lite (64-bit) which comes with no desktop environment*
-   **Storage:** (Official) Raspberry Pi Flash Drive 256GB


<a id="orgeb7cfce"></a>

# Prerequisites


<a id="orga4cc88b"></a>

## Docker

[Docker](https://www.docker.com/) is a free and open-source software that automates the deployment and running of containerized applications.

Installing Docker:

    curl -fsSL https://get.docker.com | sh

Testing docker for correct operation:

    sudo docker run --rm hello-world

Adding the user to the docker group:

    sudo usermod -aG docker <my-username> # where my-username is the username of your account

Then exit and log back in to be added to the docker group.


<a id="org346097c"></a>

## Tailscale

[Tailscale](https://tailscale.com/) is a free and open-source VPN service that allows you to connect all your devices. The free plan supports up to 100 devices and 3 users.

Installing Tailscale:

    curl -fsSL https://tailscale.com/install.sh | sh

Starting Tailscale:

    sudo tailscale up


<a id="orgdc818e0"></a>

# Services


<a id="org4d7e6ee"></a>

## Portainer

[Portainer](https://github.com/portainer/portainer) is a web app which allows you to easily organize and manage all your containers.

To install Portainer I followed this [guide](https://raspberrytips.com/portainer-on-raspberry-pi/).

Pulling the container:

    docker pull portainer/portainer-ce:latest

Starting Portainer using docker:

    sudo docker run -d -p 9000:9000 --name=portainer_app --restart=always -v /var/run/docker.sock:/var/run/docker.sock -v portainer_data:/data portainer/portainer-ce:latest

Then Portainer can be accessed at `http://<my-hostname>:9000`, where my-hostname is the hostname of your Raspberry Pi.


<a id="orga3f5683"></a>

## Nextcloud

[Nextcloud](https://nextcloud.com/) is a self-hosted productivity platform allowing you to sync your files, calendar, etc. Basically, it can replace the whole Google ecosystem (except the email service).

To install Nextcloud I followed this [guide](https://www.youtube.com/watch?v=CHWHQFwxFcE) on YouTube.

Pulling the container:

    docker pull nextcloud

Creating a docker network:

    docker network create --driver bridge nextcloud-net

Docker&rsquo;s default database is SQLite, but it is not recommended for extensive use. So, Postgres is used instead.

Starting the database (Postgres) container:

    docker run --name postgres -e POSTGRES_PASSWORD=<my-password> --network nextcloud-net -d postgres # where my-password is the password for the database

Running the Nextcloud container:

    docker run --name nextcloud -d -p 8080:80 -v /home/pi/nextcloud:/var/www/html --network nextcloud-net nextcloud

Nextcloud can be accessed at `https://<my-hostname>:8080`, where you will be prompted to create an account.

For Data folder use `/var/www/html/data`

Choose PostgreSQL for database and use `postgres` for username, database name, and container name.

Use the password that you configured for the database when you ran the container.


<a id="org685d0c0"></a>

## Nextcloud Whiteboard

This is the official [Whiteboard app](https://github.com/nextcloud/whiteboard) for Nextcloud. It is an integration of [Excalidraw](https://github.com/excalidraw/excalidraw) into Nextcloud, and supports drawing, writing text, etc. on an infinite canvas featuring real-time collaboration. I have found it to be the best self-hosting solution for hand-written notes on iPad. The stylus support is excellent and the files are being updated instantly. There have been complaints that palm rejection on the iPad does not work so well, but I have not experienced any issues so far.

To install it I just followed the instructions on Whiteboard&rsquo;s [Github](https://github.com/excalidraw/excalidraw).

Running the container:

    docker run -d --name nextcloud-whiteboard -e JWT_SECRET_KEY=<some-random-secret> -e NEXTCLOUD_URL=http://<my-hostname>:8080 -p 3002:3002 --restart unless-stopped ghcr.io/nextcloud-releases/whiteboard:stable # where some-random-secret a password of your choice, it will be needed later

Once the container is up and running, you can create a new whiteboard by going to Files > New > New whiteboard.

You will notice there is a red WiFi icon on the bottom right indicating you are offline. Indeed, the file cannot be shared among other devices, because the WebSocket server has not been configured yet. In order to configure it, you go to Administration Settings > Administration (Section) > Whiteboard. There you need to fill in the following information:

**WebSocket URL:** `http://<my-hotname>:3002`

**Shared Secret:** `<some-random-secret>`

After saving the settings, you should see a confirmation message, and you are good to go!


<a id="orgafa548e"></a>

## Bitwarden

[Bitwarden](https://bitwarden.com/) is a free and open-source password manager that allows you to store all of your passwords securely, and it is available on all platforms. Unfortunately, there is no official support for ARM-based server (yet). [Vaultwarden](https://github.com/dani-garcia/vaultwarden) is an alternative server implementation, which requires minimal resources, and it can easily be deployed on Raspberry Pi.

To install Vaultwarden I followed this [guide](https://raspberrytips.com/install-bitwarden-on-raspberry-pi/).

Pulling the container:

    docker pull vaultwarden/server:latest

Using Bitwarden/Vaultwarden requires HTTPS. So, I followed this [guide](https://www.xda-developers.com/enabled-https-secure-self-hosted-apps-tailscale/) to generate a free TLS Certificate using Tailscale. Note that the certificates are valid for 90 days.

First, enable HTTPS in the settings. This can be found in the DNS tab of the admin console.

Then, on the Raspberry Pi generate the certificate using the following command:

    sudo tailscale cert <machine-name>.<tailnet-name>.ts.net # machine-name is Raspberry's hostname by default, tailnet-name can be modified to make it easier to remember

You should see two files: `.cert` and `.key`

Transfer the generated files to the `/ssl/keys` folder.

The Vaultwarden container can finally be deployed to support HTTPS, by passing the `.key` and `.cert` files to &ldquo;Rocket&rdquo;:

    docker run -d -e ROCKET_TLS='{certs="/ssl/<machine-name>.<tailnet-name>.ts.net.cert",key="/ssl/<machine-name>.<tailnet-name>.ts.net.cert"}' -v /ssl/keys/:/ssl/ -v /vw-data/:/data/ -p 443:80 vaultwarden/server:latest


<a id="orgb126edf"></a>

## FreshRSS


<a id="org323cf64"></a>

# Backups

I am taking backups by creating snapshots using [rsnapshot](https://rsnapshot.org/). This is a snapshot utility based on [rsync](https://linux.die.net/man/1/rsync), and it can be used to take periodic backups of local or remote machines over ssh. It is making use of hard links, in order to reduce disk space usage.

To configure rsnapshot I followed this [guide](https://raspberrytips.com/rsnapshot-raspberry-pi/).

Once rsnapshot is installed, it can be configured by editing `/etc/rsnapshot.conf`.

The first step is to set the &ldquo;root&rdquo; directory, where the snapshots are going to be saved:

    snapshot_root /media/angel/raspberry_backups

In this case, I am using an external drive which has been mounted to `/media/angel`.

The next step is to set the number of backups to keep per backup level. Here I have replaced the following lines

    retain    alpha    6
    retain    beta     7
    retain    gamma    4

with those:

    retain    daily      7
    retain    weekly     4
    retain    monthly    6

The daily, weekly, and monthly correspond to the levels of backup, while the numbers represent the number of snapshots that will be maintained before the first backup is replaced.

Lastly, we can select the folders we want to backup:

    ###############################
    ### BACKUP POINTS / SCRIPTS ###
    ###############################
    
    # LOCALHOST
    backup	/home/		localhost/
    backup	/etc/		localhost/
    backup	/usr/local/	localhost/
    backup	/var/lib/	localhost/
    backup	/opt/		localhost/

Once the necessary changes have been made, the configuration can be tested/validated using the following command:

    sudo rsnapshot configtest

If the output is &ldquo;Syntax OK&rdquo;, it means the configuration does not have any syntax errors.

Before taking your first backup, you can verify that all permissions are configured correctly by running rsnapshot in dry-run mode:

    sudo rsnapshot -t daily

Finally, you can take backups by using the same command without the -t flag:

    sudo rsnapshot daily

The backups can be automated using [Cron](https://en.wikipedia.org/wiki/Cron). I have not done this yet, but if you are interested in configuring it you can follow the guide I mentioned earlier.

