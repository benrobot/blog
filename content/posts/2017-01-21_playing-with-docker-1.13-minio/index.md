---
title: "Playing with Docker 1.13 & Minio"
author: "Russ Mckendrick"
date: 2017-01-21T20:50:07.458Z
lastmod: 2021-07-31T12:34:50+01:00

tags:
 - Docker
 - DevOps
 - Minio
 - Cloud Storage
 - Docker Swarm

cover:
    image: "/img/2017-01-21_playing-with-docker-1.13-minio_0.png" 
images:
 - "/img/2017-01-21_playing-with-docker-1.13-minio_0.png"
 - "/img/2017-01-21_playing-with-docker-1.13-minio_1.png"
 - "/img/2017-01-21_playing-with-docker-1.13-minio_2.png"
 - "/img/2017-01-21_playing-with-docker-1.13-minio_3.png"
 - "/img/2017-01-21_playing-with-docker-1.13-minio_4.png"
 - "/img/2017-01-21_playing-with-docker-1.13-minio_5.png"
 - "/img/2017-01-21_playing-with-docker-1.13-minio_6.png"
 - "/img/2017-01-21_playing-with-docker-1.13-minio_7.png"
 - "/img/2017-01-21_playing-with-docker-1.13-minio_8.png"
 - "/img/2017-01-21_playing-with-docker-1.13-minio_9.png"
 - "/img/2017-01-21_playing-with-docker-1.13-minio_10.png"
 - "/img/2017-01-21_playing-with-docker-1.13-minio_11.png"
 - "/img/2017-01-21_playing-with-docker-1.13-minio_12.png"
 - "/img/2017-01-21_playing-with-docker-1.13-minio_13.png"
 - "/img/2017-01-21_playing-with-docker-1.13-minio_14.png"
 - "/img/2017-01-21_playing-with-docker-1.13-minio_15.png"


aliases:
- "/playing-with-docker-1-13-minio-9b38a36e88b"

---

I had some time today, so I thought I would combine two posts I have been planning on writing for a week or two, looking at Docker 1.13 and also Minio.

It is important to note before you read further that this post does not cover using Mino as a storage backend for containers. It is about launching a distributed Minio installation in a Docker Swarm cluster and consuming the storage using the Minio command line client and web interface.

### Docker 1.13

Docker 1.13 was released on Wednesday; you can get an overview of the features added below;

The headline feature for me was that you are now able to launch a stack using a Docker Compose file. I last looked at doing this using Distributed Application Bundles back in June last year;

[Docker Load Balancing & Application Bundles](https://media-glass.es/docker-load-balancing-application-bundles-38426d9568d2 "https://media-glass.es/docker-load-balancing-application-bundles-38426d9568d2")

It was an interesting idea but felt a little clunky, likewise, launching individual Services manually seemed a like a retrograde step.

### Launching a Docker Swarm Cluster

Before creating the cluster I needed to start some Docker hosts, I decided to launch three of them in [DigitalOcean](https://m.do.co/c/52ec4dc3647e) using the following commands;

```
docker-machine create \
--driver digitalocean \
--digitalocean-access-token your-do-token-goes-here \
--digitalocean-region lon1 \
--digitalocean-size 2gb \
swarm01

docker-machine create \
--driver digitalocean \
--digitalocean-access-token your-do-token-goes-here \
--digitalocean-region lon1 \
--digitalocean-size 2gb \
swarm02

docker-machine create \
--driver digitalocean \
--digitalocean-access-token your-do-token-goes-here \
--digitalocean-region lon1 \
--digitalocean-size 2gb \
swarm03
```

Once I had my three hosts up and running I ran the following commands to create the Swarm;

```
docker $(docker-machine config swarm01) swarm init --advertise-addr $(docker-machine ip swarm01):2377 --listen-addr $(docker-machine ip swarm01):2377
SWMTKN=$(docker $(docker-machine config swarm01) swarm join-token -q worker)
docker $(docker-machine config swarm02) swarm join $(docker-machine ip swarm01):2377 --token $SWMTKN
docker $(docker-machine config swarm03) swarm join $(docker-machine ip swarm01):2377 --token $SWMTKN
```

Then the following to check that I have the three hosts in the cluster;

```
docker $(docker-machine config swarm01) node ls
```

![](/img/2017-01-21_playing-with-docker-1.13-minio_1.png)

### Planning the Minio deployment

Before I go into any detail about launching the stack, I should talk a little about the software I will be launching. Minio describes itself as;

> A distributed object storage server built for cloud applications and devops.

There are a few buzz words in that simple description, so let’s break it down a little further. Minio provides object storage as both a distributed and standalone service. It provides a compatibility layer with the Amazon S3 v4 API meaning that any applications which have been written to take advantage of Amazon S3 should work with minimal changes, such as the endpoint to the application is connecting, to the code.

Minio also has support for [Erasure Code](https://en.wikipedia.org/wiki/Erasure_code) & [Bitrot Protection](https://en.wikipedia.org/wiki/Data_degradation#Decay_of_storage_media) built in meaning your data possesses a good level of protection when you distribute it over several drives.

For my deployment, I am going to launch six Minio containers across my three Docker hosts. As I will be attaching a local volume to each container, I will be binding them to a single host within the cluster.

While this deploy will give me distributed storage, I will only be able to lose a single container — but this is in no way production ready, so I am not too bothered about that for this post.

For more information on the Minio server compoent I will be launching see the following video;

Or go to the Minio website;

[Minio](https://minio.io "https://minio.io")

### Creating the Stack

Now I had my Docker Swarm cluster and an idea of how I wanted my application deployment to look it was time to launch a stack.

To do this, I created a Docker Compose V3 file which I based on the Minio example Docker Compose file. The compose file below is what I ended up with;

```
version: "3"

services:

  minio1:
    cover:
    image: minio/minio
    ports:
      - "9000:9000"
    networks:
      - minionet
    volumes:
      - minio1:/myexport
    environment:
      MINIO_ACCESS_KEY: 6b4535c9d0545e036d5a
      MINIO_SECRET_KEY: f50a73124f5699570beb9ad44cd940
    command: server http://minio1/myexport http://minio2/myexport http://minio3/myexport http://minio4/myexport http://minio5/myexport http://minio6/myexport
    deploy:
      restart_policy:
        condition: on-failure
      placement:
        constraints:
          - node.hostname == swarm01

  minio2:
    cover:
    image: minio/minio
    ports:
      - "9000"
    networks:
      - minionet
    volumes:
      - minio2:/myexport
    environment:
      MINIO_ACCESS_KEY: 6b4535c9d0545e036d5a
      MINIO_SECRET_KEY: f50a73124f5699570beb9ad44cd940
    command: server http://minio1/myexport http://minio2/myexport http://minio3/myexport http://minio4/myexport http://minio5/myexport http://minio6/myexport
    deploy:
      restart_policy:
        condition: on-failure
      placement:
        constraints:
          - node.hostname == swarm01

  minio3:
    cover:
    image: minio/minio
    ports:
      - "9000"
    networks:
      - minionet
    volumes:
      - minio3:/myexport
    environment:
      MINIO_ACCESS_KEY: 6b4535c9d0545e036d5a
      MINIO_SECRET_KEY: f50a73124f5699570beb9ad44cd940
    command: server http://minio1/myexport http://minio2/myexport http://minio3/myexport http://minio4/myexport http://minio5/myexport http://minio6/myexport
    deploy:
      restart_policy:
        condition: on-failure
      placement:
        constraints:
          - node.hostname == swarm02

  minio4:
    cover:
    image: minio/minio
    ports:
      - "9000"
    networks:
      - minionet
    volumes:
      - minio4:/myexport
    environment:
      MINIO_ACCESS_KEY: 6b4535c9d0545e036d5a
      MINIO_SECRET_KEY: f50a73124f5699570beb9ad44cd940
    command: server http://minio1/myexport http://minio2/myexport http://minio3/myexport http://minio4/myexport http://minio5/myexport http://minio6/myexport
    deploy:
      restart_policy:
        condition: on-failure
      placement:
        constraints:
          - node.hostname == swarm02

  minio5:
    cover:
    image: minio/minio
    ports:
      - "9000"
    networks:
      - minionet
    volumes:
      - minio5:/myexport
    environment:
      MINIO_ACCESS_KEY: 6b4535c9d0545e036d5a
      MINIO_SECRET_KEY: f50a73124f5699570beb9ad44cd940
    command: server http://minio1/myexport http://minio2/myexport http://minio3/myexport http://minio4/myexport http://minio5/myexport http://minio6/myexport
    deploy:
      restart_policy:
        condition: on-failure
      placement:
        constraints:
          - node.hostname == swarm03

  minio6:
    cover:
    image: minio/minio
    ports:
      - "9000"
    networks:
      - minionet
    volumes:
      - minio6:/myexport
    environment:
      MINIO_ACCESS_KEY: 6b4535c9d0545e036d5a
      MINIO_SECRET_KEY: f50a73124f5699570beb9ad44cd940
    command: server http://minio1/myexport http://minio2/myexport http://minio3/myexport http://minio4/myexport http://minio5/myexport http://minio6/myexport
    deploy:
      restart_policy:
        condition: on-failure
      placement:
        constraints:
          - node.hostname == swarm03

networks:
  minionet:

volumes:
  minio1:
  minio2:
  minio3:
  minio4:
  minio5:
  minio6:
```

On the face of it, the Docker Compose file seems to be very similar to a v2 file. However you may notice that there is an additional section for each of the Minio containers, everything which comes below deploy is new to the v3 Docker Compose format.

As you can see, I was able to add a restart policy and also a constraint to the file, before these options were only available if you were manually launching the service using the `docker service create`command.

If you don’t feel like using the MINIO_ACCESS_KEY and MINIO_SECRET_KEY in the Docker Compose then you can generate your own by running the following two commands which will randomly create two new keys for you;

```
openssl rand -hex 10 # for MINIO_ACCESS_KEY
openssl rand -hex 15 # for MINIO_SECRET_KEY
```

To launch the stack using this Docker Compose file, all I needed to do was the run the following command;

```
docker $(docker-machine config swarm01) stack deploy --compose-file=docker-compose.yml minio
```

![](/img/2017-01-21_playing-with-docker-1.13-minio_2.png)

As you can see from the terminal output above, six services and a network were created, as well as volumes which were created using the default volume driver.

To check that the stack was there, I ran the following;

```
docker $(docker-machine config swarm01) stack ls
```

Which gave me the output below;

![](/img/2017-01-21_playing-with-docker-1.13-minio_3.png)

Now that I knew that my Stack had successfully created and that the expected six services were attached to it, I then wanted to check that the actual services were running.

To do this, I used the following command which shows a list of all of the services attached to the specified Stack;

```
docker $(docker-machine config swarm01) stack services minio
```

As you can see from the terminal output below, I had the six services with each service having one of one containers running;

![](/img/2017-01-21_playing-with-docker-1.13-minio_4.png)

It was now time to open Minio in my browser, to do this I used the following command;

```
open http://$(docker-machine ip swarm01):9000
```

### Using Minio

Once my browser opened I was greeted with a login page;

![](/img/2017-01-21_playing-with-docker-1.13-minio_5.png)

Once logged in using the credentials I defined when launching the services, I was presented with a clean installation;

![](/img/2017-01-21_playing-with-docker-1.13-minio_6.png)

Rather than start using the web gui straight away I decided to install the command line client, I used Homebrew to do this;

```
brew install minio-mc
```

Configuration of the client was simple, all I needed to do was to run the following command;

```
mc config host add swarm http://$(docker-machine ip swarm01):9000 6b4535c9d0545e036d5a f50a73124f5699570beb9ad44cd940 S3v4
```

It added a host called swarm which can be contacted at any of the public IP addresses of my Swarm cluster (I used `swarm01`), accessed with the access & secret key and finally the host I am connecting to is running the S3v4 API.

As you can see from the terminal output below you get an overview of the which were updated or added;

![](/img/2017-01-21_playing-with-docker-1.13-minio_7.png)

Now that the client is configured I can create a Bucket and upload a file by running;

```
mc mb swarm/testing
mc cp ~/Desktop/boss-fight-free-high-quality-stock-images-photos-photography-clouds-crane-lake.jpg swarm/testing
```

I creatively called the bucket “testing”, and the file was the header graphic for this post.

![](/img/2017-01-21_playing-with-docker-1.13-minio_8.png)

Now that I had some content I can get a bucket and file listing by running;

```
mc ls swarm
mc ls swarm/testing
```

Which returned the results I expected;

![](/img/2017-01-21_playing-with-docker-1.13-minio_9.png)

Going back to the web gui I can see that the bucket and file are listed there as well;

![](/img/2017-01-21_playing-with-docker-1.13-minio_10.png)

Staying in the browser and “testing” bucket I decided to upload the Docker Compose file, to this I clicked on the **+**icon in the bottom left and then click on **Upload file**;

![](/img/2017-01-21_playing-with-docker-1.13-minio_11.png)![](/img/2017-01-21_playing-with-docker-1.13-minio_12.png)

Back on the command line check I could see the file and then used the built in `cat`command to view the contents of file;

```
mc ls swarm/testing
mc cat swarm/testing/docker-compose.yml | head -10
```

![](/img/2017-01-21_playing-with-docker-1.13-minio_13.png)

### Killing a Container

If one of the services should stop responding, I know it will get replaced as I configured each of the services with a restart condition.

To force the condition to kick in I killed one of the two containers on swarm02. First of all, I got a listing of all of the containers running on the host, and then forcibly removed it by running;

```
docker $(docker-machine config swarm02) container ls
docker $(docker-machine config swarm02) container rm -f 09d5f1e49093
docker $(docker-machine config swarm01) stack services minio
```

I checked the status of the services, and after a second or two, the replacement container had launched.

![](/img/2017-01-21_playing-with-docker-1.13-minio_14.png)

As you can see from the commands, Docker 1.13 is introducing a slightly different way to run commands which effect change to your containers rather than running `docker ps`you now run `docker container ps`.

The Docker command line client has grown quite a bit since it first launched, moving the base commands which are used to launch and interact containers underneath the container child command makes a lot of sense moving forward as it clearly identifies the part of Docker which you are controlling.

You can find a full list of the child commands at for `docker container `below;

[docker container](https://docs.docker.com/engine/reference/commandline/container/ "https://docs.docker.com/engine/reference/commandline/container/")

I would recommend that you start to use them as soon as possible as eventually the base commands originally used to control containers will be retired (probably not for quite a while, but we all have a lot of muscle memory to relearn).

### Summing up Docker 1.13

I have been waiting for the 1.13 release for a while as it is going to make working with Swarm clusters a lot more straight forward.

The introduction of clean-up commands is a godsend for managing unused volumes, networks and dangling images;

```
docker system df
docker system prune
```

![](/img/2017-01-21_playing-with-docker-1.13-minio_15.png)

Docker 1.13 also introduces a few new experimental features such as an endpoint for Prometheus-style metrics meaning that you should no longer have to use cadivisor as a middle-man, there are also options for squashing images once they are built and using compression when passing assists during a build.

I may do a follow-up post on some of these experimental features at some point soon.

### Summing up Minio

Under the hood what Minio does is quite complicated. The sign of any good software is making you forget the underlying complexities but not in a way where you can make mistakes which result in loss of service or even worse data.

Having spent hours cursing at [Ceph](https://ceph.com/) and [Swift](https://wiki.openstack.org/wiki/Swift) installations, I would recommend looking at Minio if you are after an object store which supports erasure coding out of the box.

While the example in this post is not really production ready, it isn’t too far off. I would have launched a production in an environment where I can attach external block storage directly to my containers using a volume plugin.

For example, launching a Swarm cluster in Amazon Web Services means I would be able to use Elastic Block Storage (EBS). Volumes could move alongside containers which would mean that I didn’t need to pin containers to hosts — coupled with restart conditions and more hosts I could quite quickly have fully redundant object store up and running.

Add it to your list of things to check out :)

I thought I would post a quick follow up, as always submit links to [Hacker News](https://news.ycombinator.com/submitted?id=russmck) and there was an interesting conversation in the comments. It’s well worth reading as it not only discusses the size limitation of Minio but also some of the design decisions;

[Does anyone have large (PB-scale) deployments of Minio on premises? We have a ne... | Hacker News](https://news.ycombinator.com/item?id=13452831 "https://news.ycombinator.com/item?id=13452831")