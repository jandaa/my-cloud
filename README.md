# My Cloud

This is a template of a personal cloud setup using docker swarm. It replaces many of the most common services cloud providers provide and can all be self hosted on your own personal home server.

This has a couple of advantages. Namely, that your data stays with you and is not shared with others. It is also significantly cheaper than paying a monthly subscribtion fee for cloud storage.

Setting up a server is as easy as building a cheap desktop computer or using an old one you have lying around. To this you can add in a few drives (ideally NAS specific drives in a RAID configuration) depending on your storage needs. Then you'll want to ideally install linux and connect it via ethernet to your home router.

Additionally, you'll have to purchase a domain name and point it to your home IP. This template heavily relies on using [Cloudflare](https://www.cloudflare.com) as your DNS provider but can easily be adapted to any other.

I've developed this setup over an extended period. The purpose of this repo is to hopefully save others the time and pain it took me to get setup and running.

## Services

Here is a list of the different services that are deployed with this project. For services which do not interest you, you can easily remove them from the project.

### [Nextcloud](https://nextcloud.com/)

Nextcloud is an open source alternative to google drive and iCloud. It stores and syncs your files accross devices and allows you to easily access them in one central place anywhere in the world on any device. Try it out for yourself with this [demo](https://try.nextcloud.com/).

![Alt text](doc/nextcloud.png "Nextcloud Demo")

### [Photoprism](https://photoprism.app/)

Photoprism is an open source alternative to google photos and iCloud photos. It has a really clean interface and loads extremely quickly. It also supports auto captioning and face recognition. It is by far the best open source photo solution I have seen yet and is actively being developed. Try it out for yourself with this [demo](https://demo.photoprism.app)

![Alt text](doc/photoprism.png "Photoprism Demo")

### [Jellyfin](https://jellyfin.org/)

Jellyfin is an open source alternative to video streaming services such netflix or Plex or Kodi. It allows you to store and stream your own personal media. Try it out for yourself with this [demo](https://demo.jellyfin.org/stable)

![Alt text](doc/jellyfin.png "Jellyfin Demo")

### [Wordpress](https://wordpress.org/)

Wordpress is one of the most popular content management platforms for websites. Its great for building personal or corporate websites without having to get into web development.

![Alt text](doc/wordpress.png "Wordpress Demo")

### [Nginx](https://nginx.org/en/)

Nginx is an HTTP server that can host websites and also act as a reverse proxy. In this projet, I use it as a way to host read only files that are viewable in the browser and also fetchable using commong commands like `wget` in linux.

![Alt text](doc/nginx.png "Nginx Demo")

## Setup

These step by step instructions are meant to allow anyone to quickly get setup on their personal servers. It assumes that linux is installed and that you have some minimal experience with networking and docker.

### Domain & DNS

This project assumes that you're using Cloudflare as your DNS provider. This is because it uses it to dynamically update your DNS whenever your home IP changes. It also uses the generated certificates.

Once your domain has been added to Cloudflare, you'll have to generate new CNAME records with the following subdomains: www, drive, photos, watch, files. It should look like this:

![Alt text](doc/cloudflare.png "DNS setup")

**Note:** that it may take some time for these settings to propogate. Sometimes this takes as long as a whole day.

### Setup Security & Generating certificates

Go to the SSL/TLS tab and set your encrption mode to **Full (strict)**. Then go to origin server and click **Create Certificate**. Use the PEM format and copy the certificate into a text file with the the format `<domain>.cert` where `<domain>` is the root name of your domain (e.g. for mycloud.com the file would be called mycloud.cert). Now do the same thing for the key and store that in a file `<domain>.key`. Keep these safe somewhere, we will need them later.

### Get API key

To enable Dynamic DNS updates, we will use the cloudflare web api. For this you just need to get your api key. To do this, go to your cloudflare dashboard, in the top right click on the account icon, then on **My Profile**. Go to the **API Tokens** tab and click **Create Token**. Name it cloudflare-ddns and set the permissions to the following:

![Alt text](doc/cloudflare_api_token.png "Cloudflare API token")

Copy this key to a textfile called `cloudflare_api_key` and keep it somewhere safe for now.

### Port Forwarding

In order for the outside world to connect with your server, you have to forward ports from your router to your server. This can be done by logging into your router, going to port forwarding and forwarding the ports 443 and 80 to your server.

### Update Tags

Its good practice to always set the versions of your services so that they do not change between deployments. Depending on when you come across this project, it maybe that the images are vastly out of date. If that is the case, look up the latest version and replace the version tags on the images in the docker-compose file in the format `image:<tag>`.

### Set Your Domain

Inside of the docker-compose.yml file, you'll need to do a find & replace to insert your own domain name. Replace the `<domain>` and `<com>` tags with your domain of the form `<domain>.<com>`.

### Setting up Docker Swarm

Although docker swarm comes standard with docker, it does not come initalized. To do this is easy, simply type the following command into any terminal:

```bash
docker swarm init
```

This will set your server as both the manager and worker node by default.

### Setting up a Network Interface

To communitcate between the outside world and our swarm stack, we need to make a seperate network interface with an overlay driver and call this interface `web`. To do this run the following command: 

```bash
docker network create --driver overlay web
```

### Generating Secrets

For security reasons, we don't want to put sensitive data directly into our docker-compose file (or store it on our server). Instead the common practice is to use docker secrets which stores these securly and plugs them into containers when they're deployed. 

These secrets are already setup in the docker-compose file, we just have to create them on the machine. The secrets that we have to create are:

- `<domain>`.cert
- `<domain>`.key
- clouflare_api_key
- nextcloud_db
- nextcloud_db_user
- nextcloud_db_password
- wordpress_db
- wordpress_db_user
- wordpress_db_password

**Note:** Currently photoprism does not support secrets for its admin password. This is has been submitted as a [feature request](https://github.com/photoprism/photoprism/issues/1987). For now you'll have to replace `<photoprism_admin_password>` with your intented password

There are many ways to generate secrets. One way is to store your information inside of appropriately named text files and load the secret like so:

```bash
docker secret create my-secret-name my-secrete-name.txt
```

Or you can pip the information directly from the command line like so:

```bash
echo ***** | docker secret create my-secret-name -
```

There are a few security conerns though. If you're using files you'll want to remove them from the host machine and store them elsewhere. If you're piping from the command line you'll want to delete your bash history.


### Create folders

We want all the data in our cloud to persist even when containers go down. This is accomplished using bind mounts in the docker-compose file. They mount folders in this directory to the inside of the containers. For this to work we have to make these directories. Fortunately I've included a convenince script that you can run:

```bash
bash init.sh
```

### Run

This is the last step. Congradualations for making it this far, this last step is by far the easiest and most rewarding (assuming everything else is functioning correctly). To bring up your new cloud, simply run the following:

```bash
docker swarm deploy -c docker-compose.yml --with-registry-auth mycloud
```

## Performance Tips

if you have a faster solid state drive, you can store the databases on them instead of the slower NAS drives. This should make a big impact on loading times. You can do this by changing the volume bindings of the database services to point to folders on the solid state drive.

