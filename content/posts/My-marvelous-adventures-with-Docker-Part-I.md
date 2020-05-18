---
title: "My marvelous adventures with Docker - Part I "
date: 2016-09-05T21:24:07+02:00
draft: false
---

**In this post I talk about how to get started in Docker. First we will set a Docker server, afterwards we will create your first Docker image and we will run it.**

Let's play with Docker and learn something about it!

But wait, what is Docker? [Docker](https://www.docker.com) is an open platform to build, ship and run distributed applications anywhere. So you can run apps which "live" inside a contaniner runned and mannaged by Docker.

It is very useful when you want to scale your app, the more load, the more containers containing the same app image you will deploy very quickly. It could be like Amazon Auto Scaling Group but you will not deploy full machines.

The containers run images with your apps, in this post I will create 3 images (I will call them Archons). We will store the docker images into our [Docker Hub Repo](https://hub.docker.com).

**Installing Docker**

I'm going to create an 20$/month Ubuntu Droplet in my [Digital Ocean](https://www.digitalocean.com) account and I will use [Ansible](https://www.ansible.com) to install Docker.

So, let's clone my [Sysadmin Scripts](https://github.com/a-castellano/Sysadmin-Scripts.git) repo and I will use my Docker Role to install all the necesary packages in my droplet.
``` bash
git clone https://github.com/a-castellano/Sysadmin-Scripts.git
cd Sysadmin-Scripts/Ansible/
```

Place your droplet IP into *inventory/my.hosts*.
``` ansible
[Docker]
YOUR_DROPLET_IP
```

If your is Ubuntu version is 16.04 you'll have to install python.
``` bash
ssh root@YOUR_DROPLET_IP apt-get install -y python
```

Launch the playbook and reboot your droplet after the deployment.
``` bash
ansible-playbook playbooks/setup_docker/setup.yml
```

**Deploying dockers**

We will start creating our first Archon which will contain an Apache web server. We will create a container from the [official Debian Docker image](https://hub.docker.com/_/debian).

``` bash
docker create  --name ArchonWeb --hostname ArchonWeb -i debian
Unable to find image 'debian:latest' locally
latest: Pulling from library/debian

8ad8b3f87b37: Pull complete
Digest: sha256:2340a704d1f8f9ecb51c24d9cbce9f5ecd301b6b8ea1ca5eaba9edee46a2436d
Status: Downloaded newer image for debian:latest
584c54e123ec442b9c17a1645a9057406c67d8c80f72903a377f904ce9ae7269

```

Docker checks if there exists an image called "debian" in [Docker Hub](https://hub.docker.com). If Docker doesn't have the latest version it will donwload it. When Docker gets the image it will create the container with debian inside.

``` bash
docker ps -a
CONTAINER ID        IMAGE               COMMAND             CREATED             STATUS              PORTS               NAMES
584c54e123ec        debian              "/bin/bash"         2 minutes ago       Created                                 ArchonWeb
```

Start the docker
``` bash
docker start ArchonWeb
```

``` bash
docker ps -a
CONTAINER ID        IMAGE               COMMAND             CREATED             STATUS              PORTS               NAMES
584c54e123ec        debian              "/bin/bash"         10 minutes ago      Up 6 seconds                            ArchonWeb
```

Ok, our first docker is alive, let's enter into it.

``` bash
docker exec -i -t ArchonWeb  /bin/bash
```

Congratulations! We are inside our first Docker container running!

As ArchonWeb will be a web server we are going to install Apache2.

``` bash
apt-get update && apt-get upgrade -y && apt-get install apache2 apache2-utils php5 php5-mcrypt php5-mysql php5-cli php5-common php5-json php5-readline php-pear libmcrypt4 libapache2-mod-php5 libmcrypt-dev mcrypt mariadb-client net-tools -y
service apache2 start
```

Apache2 is running!

``` bash
netstat -punta | grep LISTEN
tcp6       0      0 :::80                   :::*                    LISTEN      3488/apache2
```

Wait, Apache2 is listening on a private address so we can't connect to this server. Don't start to cry yet, we can solve this issue.

We are making changes in our machine therefore our first step will be create a new image from "debian" one.

Let's exit from our docker.

``` bash
exit
```

Stop the Docker

``` bash
docker stop ArchonWeb
```

Commit a new image called ArchonWeb from the original debian one.

``` bash
docker commit -m "Creating my first image" ArchonWeb
```

Now, we have two images in our machine.

``` bash
docker images
REPOSITORY          TAG                 IMAGE ID            CREATED             SIZE
<none>              <none>              YOUR_IMAGE_ID       57 seconds ago      262.4 MB
debian              latest              031143c1c662        3 days ago          125.1 MB
```

Ok, this is not beauty and it would be better to have our new image in a repository. Create and account in [Docker Repo](https://hub.docker.com) and create a new repo too.

![Creating the repo](/images/creating_repo.png)

Log to your Docker Hub account.

``` bash
docker login
Login with your Docker ID to push and pull images from Docker Hub. If you don't have a Docker ID, head over to https://hub.docker.com to create one.
Username: YOUR_USERNAME
Password: YOUR_PASSWORD
Login Succeeded
```

``` bash
docker commit -m "Creating my first image" ArchonWeb YOUR_USERNAME/archonweb
```


To probe that all is ok let's delete our container and all the images

``` bash
root@Docker:~|⇒  docker rm ArchonWeb
ArchonWeb
root@Docker:~|⇒  docker images
REPOSITORY              TAG                 IMAGE ID            CREATED             SIZE
YOUR_USERNAME/archonweb   latest              c739c75c3b8e        33 minutes ago      262.4 MB
<none>                  <none>              ad6d1440ebde        40 minutes ago      262.4 MB
debian                  latest              031143c1c662        3 days ago          125.1 MB
root@Docker:~|⇒  docker rmi c739c75c3b8e ad6d1440ebde 031143c1c662
Untagged: YOUR_USERNAME/archonweb:latest
Untagged: YOUR_USERNAME/archonweb@sha256:8e1e908fe823ef1b746718ebb7564548f4f0ea076faaae733f667fb5946acee7
Deleted: sha256:c739c75c3b8e690427a845ae79206b1e3037fddf604765dc5e0478536161e5bb
Deleted: sha256:ad6d1440ebde55617462269619cd457bfa825d8f307c126d7fbc0ca3201c493f
Deleted: sha256:a6a314f5c1a5da3d06226fe40216731384d3f80fafade48a2b1cef3f07384b96
Untagged: debian:latest
Untagged: debian@sha256:2340a704d1f8f9ecb51c24d9cbce9f5ecd301b6b8ea1ca5eaba9edee46a2436d
Deleted: sha256:031143c1c662878cf5be0099ff759dd219f907a22113eb60241251d29344bb96
Deleted: sha256:9e63c5bce4585dd7038d830a1f1f4e44cb1a1515b00e620ac718e934b484c938
```

Now we are going to create a new container with our own image and we will redirect the host port 8080 to our docker 80 port.

``` bash
docker create  --name ArchonWeb --hostname ArchonWeb -p 8080:80 -i YOUR_USERNAME/archonweb
docker start ArchonWeb
```

Start the container, enter into it and start the Apache2 web server. Now go to your docker host with port 8080 and you'll see the Apache default page.


![Apache 2 Default Server](/images/apache2_default_page.png)

You can configure the container to launch apache2 when you start the container.

```bash
docker create --name ArchonWeb --hostname ArchonWeb -p 8080:80 -i YOUR_USERNAME/archonweb /usr/sbin/apache2ctl -D FOREGROUND
docker start ArchonWeb
```

If you followed all the instructions until here you have ArchonWeb running. Congratulations!

Probably you are having some questions and concerns such as "What happens if we put a private webpage in our container, will it be public if we commit our images? Where is the database for my webapps? No worries, this was a first example of how Docker works. In the following posts of this series I will address all these questions and more. For starters, in the next adventure I will talk about **volumes** and **Docker Composer** (for automatize your dockers deployment).
