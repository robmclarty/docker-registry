# How to Make a Simple Private Registry for Docker Images

If you're using Docker in production you've probably encountered the problem
of trying to determine where to store your images. Sometimes a simple solution
is in order, such as simply SSHing them onto the host machine directly. But
oftentimes you want something more robust, more secure, more automated, and
thus more cloudy.

Many of the big boys and girls have offerings that will do these things for you,
for money, and arguably more easily. However, if you want total control and
accountability and/or would rather spend a bit of time over a bit of money, you
can roll your own without too much headache.

Here's how.


## 1. Create New EC2 Instance (or other machine/vm/whathaveyou)

Create folders for shared volumes.

```
mkdir ~/nginx
mkdir ~/data
mkdir ~/certs
```

I like using these directories personally, but feel free to organize your host
machine as you like. I put my nginx configs (and `htpasswd`) in `~/nginx`, any
app-related data (e.g., perhaps an app-specific crypto key file, or config) in
`~/data`, and  finally, my SSL/TLS certs in `~/certs`. I use these directories
from my home folder simply for convenience.


## 2. Install Apache Utils

So we can use http basic auth. You could setup something more sophisticated if
you're so inclined, but this does the job (over TLS of course) to simply grab
an image and pull it down.

`sudo apt-get -y install apache2-utils`

- [Private Docker Registry with Letsencrypt](https://van-rossum.com/2016/12/07/Private-docker-registry-with-letsencrypt.html)


## 3. Install Docker

Docker is available for multiple platforms. Follow the instructions for your
platform of choice. In this article I will be demonstrating an installation
on Ubuntu Linux.

- [How to Install Docker](https://docs.docker.com/engine/installation/)
- [How to Install Docker on Ubuntu](https://docs.docker.com/engine/installation/linux/ubuntu/)

### Install dependencies

Necessary tools for installing Docker.

```
sudo apt-get install \
  apt-transport-https \
  ca-certificates \
  curl \
  software-properties-common
```

### Add Docker's official GPG key

`curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo apt-key add -`


### Verify that the key fingerprint is correct

Should be `9DC8 5822 9FC7 DD38 854A E2D8 8D81 803C 0EBF CD88`

```
$ sudo apt-key fingerprint 0EBFCD88

pub   4096R/0EBFCD88 2017-02-22
      Key fingerprint = 9DC8 5822 9FC7 DD38 854A  E2D8 8D81 803C 0EBF CD88
uid                  Docker Release (CE deb) <docker@docker.com>
sub   4096R/F273FCD8 2017-02-22
```

### Add Docker repository from stable branch

```
sudo add-apt-repository \
  "deb [arch=amd64] https://download.docker.com/linux/ubuntu \
  $(lsb_release -cs) \
  stable"
```

### Install Docker

```
sudo apt-get update
sudo apt-get -y install docker-ce
```


### Install Docker-compose

```
sudo -i
curl -L https://github.com/docker/compose/releases/download/1.13.0/docker-compose-`uname -s`-`uname -m` > /usr/local/bin/docker-compose
chmod +x /usr/local/bin/docker-compose
```


## 4. Install Letsencrypt with Certbot

Install the EFF's Certbot script. This script will auto-detect the environment
of your server and use that information to setup
[Letsencrypt](https://letsencrypt.org/) for your system automatically.

```
sudo ~/certbot-auto certonly \
  --standalone \
  --email admin@example.com \
  -d registry.domain.com
```

This will create 4 files located in `/etc/letsencrypt/live/myregistry.domain.com/`.

- **privkey.pem** - private key for cert
- **fullchain.pem** - cert.pem + chain.pem
- **cert.pem** - server cert
- **chain.pem** - server cert + intermediate cert

In the `~/certs` folder, create a dhparam key.

`sudo openssl dhparam -out ~/certs/dhparam.pem 4096`

Finally, setup a cronjob to auto-renew the certs.

`sudo crontab -e`

..and add this line:

`48 3,15 * * * ~/certbot-auto renew >> /var/log/letsencrypt-renew.log"`

This renews the certs at 3:48am and 3:48pm every day. Letsencrypt will only
actually renew the certs when they need to be (e.g., every month).

- [Certbot Source](https://github.com/certbot/certbot)
- [How to Install Certbot](https://certbot.eff.org/docs/install.html)


## 5. Copy Config Files

Copy your Compose file(s) and server config files to the host machine.

```
scp ./docker-compose.yml username@host:~/
scp ./docker-compose-prod.yml username@host:~/
scp ./nginx/registry.conf username@host:~/nginx
```

For example config files used for a similar registry machine, see my sample
[docker-registry](https://github.com/robmclarty/docker-registry) repo on Github.


## 7. Setup Basic Auth

`htpasswd -c ~/nginx/htpasswd myUsername`

This will prompt you for a password and store it, along with the username, in
the file `/path/to/htpasswd` using bcrypt to hash the password before writing it.


## 8. Startup Docker Containers with Compose

`sudo docker-compose -f docker-compose.yml -f docker-compose-prod.yml up -d`

Use the `-f` flag to load multiple compose files, later files overriding
earlier files.

To stop the daemon (because we started docker with the `-d` flag), use the
`down` command:

`sudo docker-compose down --rmi 'all'`


## 9. Login from Your Local Machine

`docker login https://myregistry.domain.com -u myUsername -p myPassword`

Your config for registries you've logged into will be stored in `~/.docker/config.json`.


## 10. Tag and Push an Image

`docker tag myImage myregistry.domain.com/myImage`

Alternatively, you can build + tag (even multiple tags) in a single command:

`docker build -t myregistry.domain.com/myImage:0.12 -t myregistry.domain.com/myImage:latest .`

Finally, simply issue the `push` command referencing the image on your local
machine (omit tags to push all matching images).

`docker push myregistry.domain.com/myImage`


## 11. Pull an Image

`docker pull myregistry.domain.com/myImage`


## Dance

Now that you have a cloud-based private Docker repository you can do cool stuff
like spin up new containers from Compose on other remote machines which
reference your images using your new registry's URL. This is exactly what you
want for automating everything such that your orchestration system can tear
down and spin up new containers on the fly, auto-scale, and (re)build without
the need for human intervention. But it also makes deployments easier too since
all you need to do is push your image to the repo, and then just tell your
servers to update themselves ;)


## References

- [Docker Registry HTTP API V2](https://github.com/docker/distribution/blob/master/docs/spec/api.md)
- [How to Setup a Private Docker Registry on Ubuntu 14.04](https://www.digitalocean.com/community/tutorials/how-to-set-up-a-private-docker-registry-on-ubuntu-14-04)
