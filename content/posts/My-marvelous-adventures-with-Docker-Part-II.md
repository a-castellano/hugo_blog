---
title: "My marvelous adventures with Docker - Part II "
date: 2016-09-18T21:24:07+02:00
draft: false
---

**In this post I continue my adventures with Docker. After creating our first image and container in the previous [post](https://a-castellano.github.io/posts/my-marvelous-adventures-with-docker-part-i/) I will look into how docker images can autostart services and how to establish communication between containers.**

During this post we are creating images and storing them in Docker Hub. All my images are stored in my [ArchonWeb](https://hub.docker.com/r/acastellano/archonweb/) repository. All the scripts used in this post an be found in my [Docker repo](https://github.com/a-castellano/docker).

First of all we should remove all containers and images in order to maintain our workspace clean.

``` bash
docker stop $(docker ps -a -q)
docker rm $(docker ps -a -q)
docker rmi $(docker images -q)
```

Let's create again our container.

``` bash
docker create --name ArchonWeb --hostname ArchonWeb -p 8080:80 -i YOUR_USERNAME/archonweb /usr/sbin/apache2ctl -D FOREGROUND
```

This command looks nice but... What all these arguments mean?

**--name:** The name of the container
**--hostname:** The hostname
**-p:** the port forwarding definition, write as many as you need
**-i:** the desired image to deploy

After all these arguments we have to specify which command (with its own arguments) will be executed by our container on start.

**/usr/sbin/apache2ctl -D FOREGROUND**

If we look our container properties we'll see:
``` bash
docker ps -a
CONTAINER ID        IMAGE                      COMMAND                  CREATED             STATUS              PORTS               NAMES
5f3286af690b        YOUR_USERNAME/archonweb    "/usr/sbin/apache2ctl"   12 minutes ago      Created                                 ArchonWeb
```

What happens if we don't specify the apache command?
``` bash
docker stop ArchonWeb && docker rm ArchonWeb
docker create --name ArchonWeb --hostname ArchonWeb -p 8080:80 -i YOUR_USERNAME/archonweb
```

Let's see:
``` bash
docker ps -a
CONTAINER ID        IMAGE                   COMMAND             CREATED             STATUS              PORTS               NAMES
2250cc8640ae        YOUR_USERNAME/archonweb "/bin/bash"         2 minutes ago       Created                                 ArchonWeb
```

By default our container will execute bash process. As we don't specify any argument or command, apache2 won't be called
``` bash
docker start ArchonWeb
docker ps -a
CONTAINER ID        IMAGE                   COMMAND             CREATED             STATUS              PORTS                  NAMES
2250cc8640ae        YOUR_USERNAME/archonweb "/bin/bash"         17 seconds ago      Up 9 seconds        0.0.0.0:8080->80/tcp   ArchonWeb
```

We won't see nothing in http://SERVER_IP:8080.

But what if we want our container to run apache2 by default?

We need to update our image configuration, to do that we will create a new container with our desired settings. After doing this we will create a new image.

``` bash
docker create --name ArchonWeb --hostname ArchonWeb -p 8080:80 -i YOUR_USERNAME/archonweb /usr/sbin/apache2ctl -D FOREGROUND
```

``` bash
docker ps -l
CONTAINER ID        IMAGE                   COMMAND                  CREATED             STATUS              PORTS               NAMES
ea13cfe69acf        YOUR_USERNAME/archonweb "/usr/sbin/apache2ctl"   50 seconds ago      Created                                 ArchonWeb
```

There are no ports listed because the container is not in execution.

``` bash
docker commit -m "Added COMMAND" -a "YOUR NAME" ArchonWeb YOUR_USERNAME/archonweb:v2
```

``` bash
docker images
REPOSITORY               TAG                 IMAGE ID            CREATED             SIZE
YOUR_USERNAME/archonweb  v2                  a8870e0efa0a        44 seconds ago      262.4 MB
YOUR_USERNAME/archonweb  latest              c739c75c3b8e        2 days ago          262.4 MB
```

If you are not signed in our [Docker Hub](https://hub.docker.com) account the following step will fail.
``` bash
docker push YOUR_USERNAME/archonweb:v2
```

In our [Docker Hub](https://hub.docker.com) account we could see the two tags.

![My two tags](/images/docker_image_tags01.png)

OK, let's test if all would work as we wish. We will delete all the containers and images and we will deploy our archonweb:v2 in a new container.

``` bash
docker stop $(docker ps -a -q)
docker rm $(docker ps -a -q)
docker rmi $(docker images -q)
```

``` bash
docker create --name ArchonWeb --hostname ArchonWeb -p 8080:80 -i YOUR_USERNAME/archonweb:v2 && docker ps -a
...
CONTAINER ID        IMAGE                       COMMAND                  CREATED                  STATUS              PORTS               NAMES
381681d2078c        YOUR_USERNAME/archonweb:v2  "/usr/sbin/apache2ctl"   Less than a second ago   Created                                 ArchonWeb
```

``` bash
docker inspect ArchonWeb
[
{
                "Id": "381681d2078c773ed9ca18db41ae006baa1e15c969cc7769f590f74105d8ff73",
                        "Created": "2016-09-05T19:47:41.793743389Z",
                                "Path": "/usr/sbin/apache2ctl",
                                "Args": [
                                            "-D",
                                                        "FOREGROUND"
                                                                
                                ],

}
]
```

After starting our docker container we would see our apache default page through port 8080 in our docker host.

**Using Dockerfile to create images**

In this section we are going to use **Dockerfile**. A Dockerfile is a special file which tells docker how to build images. We are going to create again our Archonweb:v2 image using a Dockerfile.

Clone my [docker repo](https://github.com/a-castellano/docker) in our /root/ folder.

``` bash
git clone https://github.com/a-castellano/docker.git
cd docker/ArchonWeb
tree
.
├── apache2_sites_enabled
│   ├── 000-default.conf
│   └── docker.my-site.io.conf
├── apache2_www
│   ├── docker.my-site.io
│   │   └── index.html
│   └── html
│       └── default
│           └── index.html
├── Dockerfile
└── README.md
```

There are more files side from the DockerFile, we will use the other later. 

This is our Dockerfile.

```
FROM acastellano/archonweb:latest
MAINTAINER Álvaro Castellano Vela <alvaro.castellano.vela@gmail.com>
RUN apt-get update -y
RUN apt-get upgrade -y
EXPOSE 80
CMD ["/usr/sbin/apache2ctl", "-D", "FOREGROUND"]
```

This Docker file will create a new image **FROM** our archonweb:latest current image, it will update our packages (using **RUN** Docker will execute commands inside our image). The Dockerfile defines the open ports that will be open too. Finally, tt sets the image command for starting apache2 when the container starts.

Build the image form using Dockerfile
``` bash
docker build -t YOUR_USERNAME/archonweb:v2 .
docker images
REPOSITORY               TAG                 IMAGE ID            CREATED             SIZE
YOUR_USERNAME/archonweb  v2                  6c732b5bae3d        45 seconds ago      276.4 MB
acastellano/archonweb    latest              c739c75c3b8e        12 days ago         262.4 MB
```

``` bash
docker create --name ArchonWeb --hostname ArchonWeb -p 8080:80 -i YOUR_USERNAME/archonweb:v2
```

If we start our new ArchonWeb container it will work in the same way as before.

As we did before we can commit the image:

``` bash
docker commit -m "Image created using a Dockerfile" -a "YOUR NAME" ArchonWeb YOUR_USERNAME/archonweb:v2
```

**Using volumes**

So, we have a web server without content. Next step would be create a website or webapp but there is a little problem. If we enter inside our container and we try to update our index.html we will realize that this contaner doesn't have any text editor!
``` bash
docker exec -i -t ArchonWeb /bin/bash
vi /var/www/html/index.html
bash: vi: command not found

nano /var/www/html/index.html
bash: nano: command not found
```

Why we don't have any editor? Don't lose your mind!

Docker images should be as small as possible so it would be reasonable to have only essential packages installed. So please, don't install text editors or other tools that are not necessary for the main purpose of the image.

We are going to use volumes. A volume is a local directory which is mounted in our container, so we can edit files that the container can see. When we mount a volume in existing location it will be overwritten.

Inside our repository there are two folders: **apache2_sites_enabled** will be mounted in **/etc/apache2/sites-enabled** and **apache2_www** will be mounted in **/var/www**.

For doing this we will use the **-v** option which allows us to attach volumes.
``` bash
docker create --name ArchonWebVolumes --hostname ArchonWebVolumes -p 8081:80 -v apache2_www:/var/www -v apache2_sites_enabled:/etc/apache2/sites-enabled -i YOUR_USERNAME/archonweb:v2
```

This might not work, ensure that we are in the folder ArchonWeb which is inside of the docker root folder.
``` bash
echo $PWD
/root/docker/ArchonWeb
```

We have to use absolute paths.
``` bash
docker create --name ArchonWebVolumes --hostname ArchonWebVolumes -p 8081:80 -v $PWD/apache2_www:/var/www -v $PWD/apache2_sites_enabled:/etc/apache2/sites-enabled -i YOUR_USERNAME/archonweb:v2
docker start ArchonWebVolumes
```

Now it is working, our default site has changed. If we could see "Nothing for now" in http://SERVER_IP:8081 congratulations! We did it!
In the same way, if we point docker.my-site.io to our SERVER_IP we could see a default web template in http://docker.my-site.io:8081

![Our new default website](/images/archonwebv2_default_website.png)
![Our docker website](/images/docker-my-site-io.png)

Of course, as we modify our default index it will change without doing anything more.

``` bash
vim $PWD/apache2_www/html/default/index.html
```

![Making changes](/images/something-for-now.png)

**Using Dockerfile for attaching volumes**

Using a Dockerfile we can go further and write instructions to attach volumes. Inside the docker repository there is another folder called ArchonWebVolumes.
``` bash
cd ~/docker/ArchonWebVolumes
```

Here is our DockerFile:
```
FROM acastellano/archonweb:v2
MAINTAINER Álvaro Castellano Vela <alvaro.castellano.vela@gmail.com>
EXPOSE 80
VOLUME /etc/apache2/sites-enabled
VOLUME /var/www
ADD apache2_sites_enabled /etc/apache2/sites-enabled
ADD apache2_www /var/www
RUN apt-get update -y && apt-get upgrade -y
ENTRYPOINT ["/usr/sbin/apache2ctl", "-D", "FOREGROUND"]
```

In this file there are two new commands: using **VOLUME** we are defining which path will be a volume. Using **ADD** we define volume relationships as we did using **-v** command. Let's build the new image.
``` bash
docker build -t archonwebvolumes .
```

Now we create a new container using the new image:
``` bash
docker create  --name ArchonWebVolumes --hostname ArchonWebVolumes -p 8082:80 -i archonwebvolumes
docker start ArchonWebVolumes
```

We will see "Nothing for now" in http://SERVER_IP:8082. As we did before we can modify the files inside the volumes.

Now imagine that inside *apache2_www* we have some ultra marvelous app, after finishing its development we want to create a new docker image containing our release version which is inside.

Let's create another image with the release version of our app inside.

``` bash
docker stop $(docker ps -a -q)
docker rm $(docker ps -a -q)
docker rmi $(docker images -q)
cd ~/docker/ArchonWeb
docker build -t YOUR_USERNAME/archonweb:release .
docker create  --name ArchonWebRelease --hostname ArchonWebRelease -p 8080:80 -i YOUR_USERNAME/archonweb:release
```

The container is ready, now it's time to copy our release version. For coping files inside our container we need to use **docker cp**. It is not necessary to start the container for copying files inside it.

``` bash
docker cp ../ArchonWebVolumes/sites-enabled ArchonWebRelease:/etc/apache2/
docker cp ../ArchonWebVolumes/www ArchonWebRelease:/var/
docker start ArchonWebRelease
```

If we could see "Nothing for now" in http://SERVER_IP:8080 we did it. Let's commit our image.

``` bash
docker commit -m "Release" -a "YOUR_NAME" ArchonWebRelease  YOUR_USERNAME/archonweb:release
docker push YOUR_USERNAME/archonweb:release
```

**Docker compose**

In the last part of this post we are going to create two containers. First container will be a new version of our ArchonWeb container. Second container will be a database server, we will use MariaDB. Instead of creating our own docker image we will use the public MariaDB public image.

Our new ArchonWeb app needs to connect to a database so we need to establish a connection between our two containers. We could create the containers and use docker for configuring a new network and connect our containers to it but there is another way to do it.

It is possible to automatize this process using [Docker Compose](https://docs.docker.com/compose/#/docker-compose).

According with Docker docs, Compose is a tool for defining and running multi-container Docker applications. We will use Compose for creating our new service.

Our first step will be setting up our new version of ArchonWeb.

``` bash
cd ~/docker/Composer_Archonweb_Mariadb
tree .
.
├── archonweb
│   ├── apache2_sites_enabled
│   │   └── 000-default.conf
│   ├── apache2_www
│   │   └── default
│   │       └── index.php
│   └── docker-entrypoint.sh
├── DockerCloudStackfile.yml
├── docker-compose.yml
└── Dockerfile

4 directories, 6 files
```

This is our web app:
**Composer_Archonweb_Index.php**
```
<html>
	<head>
		<title>Archon Web</title>
	</head>
	<body>
		<h1><?php echo "ArchonWeb" ?></h1>

		<?php 
			$dbname = '##MYSQL_DATABASE##';
			$dbuser = '##MYSQL_USER##';
			$dbpass = '##MYSQL_PASSWORD##';
			$dbhost = '##MYSQL_HOST##';

			$connect = mysql_connect($dbhost, $dbuser, $dbpass) or die("Unable to Connect to '$dbhost'");
			
			mysql_select_db($dbname) or die("Could not open the db '$dbname'");

			$test_query = "SHOW TABLES FROM $dbname";

			$result = mysql_query($test_query);

			$tblCnt = 0;

			while($tbl = mysql_fetch_array($result)) {
	  			$tblCnt++;
			}

			if (!$tblCnt) {
				echo "There are no tables<br />\n";
			} else {
				echo "There are $tblCnt tables<br />\n";
			}

		?>

	</body>
</html>
```

This app only counts how many tables are in our "##MYSQL_DATABASE##". The database values use ##'s strings because our container will set these values on start. How it will do it?

Our new ArchonWeb will execute this script on start, let's see what it does:
```
#!/bin/bash
set -eo pipefail
shopt -s nullglob

if [ -z ${MYSQL_DATABASE} ]; then
    echo "You need to specify MYSQL_DATABASE"
    exit 1
fi

if [ -z ${MYSQL_USER} ]; then
    echo "You need to specify MYSQL_USER"
    exit 1
fi

if [ -z ${MYSQL_PASSWORD} ]; then
    echo "You need to specify MYSQL_PASSWORD"
    exit 1
fi

if [ -z ${MYSQL_HOST} ]; then
    echo "You need to specify DB_HOST"
    exit 1
fi

sed -i -e "s/##MYSQL_DATABASE##/$MYSQL_DATABASE/g" /var/www/default/index.php
sed -i -e "s/##MYSQL_USER##/$MYSQL_USER/g" /var/www/default/index.php
sed -i -e "s/##MYSQL_PASSWORD##/$MYSQL_PASSWORD/g" /var/www/default/index.php
sed -i -e "s/##MYSQL_HOST##/$MYSQL_HOST/g" /var/www/default/index.php

/usr/sbin/apache2ctl -D FOREGROUND
```

When the container starts, this script checks if the required environment variables are set, after that it will modify the index.php and it will run apache2 web server.

All these files need to be copied inside our container, we will do it after creating our ArchonWeb image.

For this new version our Dockerfile is very simple.
```
FROM acastellano/archonweb:v2
MAINTAINER Álvaro Castellano Vela <alvaro.castellano.vela@gmail.com>
EXPOSE 80
```

Let's create the image and the container.
``` bash
docker stop $(docker ps -a -q)
docker rm $(docker ps -a -q)
docker rmi $(docker images -q)
cd ~/docker/Composer_Archonweb_Mariadb
docker build -t YOUR_USERNAME/archonweb:v3 .
docker create --name ArchonWeb --hostname ArchonWeb -p 8080:80 -i YOUR_USERNAME/archonweb:v3
```

Now, let's copy the web app and the startup script.
``` bash
cd ~/docker/Composer_Archonweb_Mariadb/archonweb
docker cp apache2_sites_enabled/000-default.conf ArchonWeb:/etc/apache2/sites-enabled/
docker cp apache2_www/default ArchonWeb:/var/www/
docker cp docker-entrypoint.sh ArchonWeb:/
docker commit -m "Added index.php and docker-entrypoint.sh" -a "Your Name" ArchonWeb  YOUR_USERNAME/archonweb:v3
docker push YOUR_USERNAME/archonweb:v3
```

If we start our container it won't work because there are no databases and our app is not configured.
![Our app is not configured](/images/archonwebv3_index_not_configured.png)

The required ArchonWeb version is ready. Now is time to deploy our new application. This is our Docker Compose script:
```
version: '2'
services:
    db:
        container_name: db
        image: mariadb:latest
        volumes:
         - "./.data/db:/var/lib/mysql"
        restart: always
        environment:
          MYSQL_ROOT_PASSWORD: toor
          MYSQL_DATABASE: archondb
          MYSQL_USER: archon
          MYSQL_PASSWORD: password

    ArchonWeb:
        depends_on:
          - db
        container_name: ArchonWeb
        image: acastellano/archonweb:v3
        command: /docker-entrypoint.sh
        links: 
          - db
        ports:
          - "8080:80"
        restart: always
        environment:
          MYSQL_DATABASE: archondb
          MYSQL_USER: archon
          MYSQL_PASSWORD: password
          MYSQL_HOST: db
```

This script defines two services: **db** and **ArchobWeb**. Each service needs a image to be deployed. Also, **db** container mounts a volume where our database will be stored, it won't be secure if our database is stored publically. If we commit our db image it won't contain private data.

For each service the script defines **environment** variables which will be used to configure the containers.

The **command** defines which script will be executed on start.

In the **ArchonWeb** section is defined a dependency with the **db** container, it won't be created until **db** exists. Both container will be in the same network, with the "**link**" action we can reach **db** from ArchonWeb, it is not necesaary to use **db** IP address, we can reach it only using "db" name.

Let's clean all the containers and images again and launch our docker service.
``` bash
docker stop $(docker ps -a -q)
docker rm $(docker ps -a -q)
docker rmi $(docker images -q)
cd ~/docker/Composer_Archonweb_Mariadb
docker-compose up -d
docker ps -a
CONTAINER ID        IMAGE                      COMMAND                  CREATED             STATUS                  PORTS                  NAMES
235efc8c9083        acastellano/archonweb:v3   "/docker-entrypoint.s"   1 seconds ago       Up Less than a second   0.0.0.0:8080->80/tcp   ArchonWeb
fbb272c61a73        mariadb:latest             "docker-entrypoint.sh"   1 seconds ago       Up Less than a second   3306/tcp               db
```

If our app works we could see something like this in http://SERVER_IP:8080.

![Our app is working!](/images/archonwebv3_working.png)

Let's explore our container.
``` bash
docker exec -i -t ArchonWeb /bin/bash
cat /var/www/default/index.php
```

Our app file has changed according with the environment variables.
```
<html>
	<head>
		<title>Archon Web</title>
	</head>
	<body>
		<h1><?php echo "ArchonWeb" ?></h1>

		<?php
			$dbname = 'archondb';
			$dbuser = 'archon';
			$dbpass = 'password';
			$dbhost = 'db';

			$connect = mysql_connect($dbhost, $dbuser, $dbpass) or die("Unable to Connect to '$dbhost'");

			mysql_select_db($dbname) or die("Could not open the db '$dbname'");

			$test_query = "SHOW TABLES FROM $dbname";

			$result = mysql_query($test_query);

			$tblCnt = 0;

			while($tbl = mysql_fetch_array($result)) {
	  			$tblCnt++;
			}

			if (!$tblCnt) {
				echo "There are no tables<br />\n";
			} else {
				echo "There are $tblCnt tables<br />\n";
			}

		?>

	</body>
</html>
```

Let's test if our app is working creating some tables.

Inside Archonweb:
``` bash
mysql -uarchon -ppassword -hdb archondb
```
``` bash
MariaDB [archondb]> CREATE TABLE example ( id INT, data VARCHAR(100) );
MariaDB [archondb]> CREATE TABLE example ( id INT, data VARCHAR(100) );
```

After creating two tables we will see this in http://SERVER_IP:8080:

![Two tables](/images/archonwebv3_two_tables.png)

This is it so far folks!

I'm learning new things about Docker and I will write about it soon. Stay tuned.
