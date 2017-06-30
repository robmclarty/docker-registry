# Simple Private Registry for Docker Images

- https://github.com/docker/distribution/blob/master/docs/spec/api.md
- https://www.digitalocean.com/community/tutorials/how-to-set-up-a-private-docker-registry-on-ubuntu-14-04


### 1. Create new EC2 instance (or other VM)

Create folders for shared volumes.

```
mkdir ~/nginx
mkdir ~/data
mkdir ~/certs
```


### 2. Install apache utils (so we can create htpasswd file)

- https://van-rossum.com/2016/12/07/Private-docker-registry-with-letsencrypt.html

`sudo apt-get -y install apache2-utils`


### 3. Install Docker

- https://docs.docker.com/engine/installation/linux/ubuntu/

#### Install dependencies

```
sudo apt-get install \
  apt-transport-https \
  ca-certificates \
  curl \
  software-properties-common
```

#### Add Docker's official GPG key

`curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo apt-key add -`

#### Verify that the key fingerprint is correct

Should be `9DC8 5822 9FC7 DD38 854A E2D8 8D81 803C 0EBF CD88`

```
$ sudo apt-key fingerprint 0EBFCD88

pub   4096R/0EBFCD88 2017-02-22
      Key fingerprint = 9DC8 5822 9FC7 DD38 854A  E2D8 8D81 803C 0EBF CD88
uid                  Docker Release (CE deb) <docker@docker.com>
sub   4096R/F273FCD8 2017-02-22
```

#### Add Docker repository from stable branch

```
sudo add-apt-repository \
  "deb [arch=amd64] https://download.docker.com/linux/ubuntu \
  $(lsb_release -cs) \
  stable"
```

#### Install Docker

`sudo apt-get update`
`sudo apt-get -y install docker-ce`

#### Install Docker-compose

```
sudo -i
curl -L https://github.com/docker/compose/releases/download/1.13.0/docker-compose-`uname -s`-`uname -m` > /usr/local/bin/docker-compose
chmod +x /usr/local/bin/docker-compose
```


### 4. Install Letsencrypt with Certbot

- https://github.com/certbot/certbot
- https://certbot.eff.org/docs/install.html

Install Certbot script. This script will auto-detect the environment of your
server and use that information to setup Letsencrypt for your system
automatically.

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

### 5. Copy config files

`scp ./docker-compose.yml username@host:~/`
`scp ./docker-compose-prod.yml username@host:~/`
`scp ./nginx/registry.conf username@host:~/nginx`


### 7. Setup Basic Auth

`htpasswd -c ~/nginx/htpasswd myUsername`

This will prompt you for a password and store it, along with the username, in
the file `/path/to/htpasswd` using bcrypt to hash the password before writing it.


### 8. Startup docker containers

`sudo docker-compose -f docker-compose.yml -f docker-compose-prod.yml up -d`

Use the `-f` flag to load multiple compose files, later files overriding
earlier files.

To stop the daemon (because we started docker with the `-d` flag), use the
`down` command:

`sudo docker-compose down --rmi 'all'`


### 9. Login

`docker login https://myregistry.domain.com -u myUsername -p myPassword`

Your config for registries you've logged into will be stored in `~/.docker/config.json`.


### 10. Tag and Push an Image

`docker tag myImage myregistry.domain.com/myImage`

Alternatively, you can build + tag (even multiple tags) in a single command:

`docker build -t myregistry.domain.com/myImage:0.12 -t myregistry.domain.com/myImage:latest .`

Finally, simply issue the `push` command referencing the image on your local
machine (omit tags to push all matching images).

`docker push myregistry.domain.com/myImage`


### 11. Pull an Image

`docker pull myregistry.domain.com/myImage`
