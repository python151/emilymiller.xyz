+++
author = "Emily Miller"
title = "How to host a CTF"
date = "2025-08-17"
description = "A crash course in CTF infrastructure"
categories = [ "misc" ]
+++

# Part 0: Infrastructure

In order to run this, you'll need some sort of server (a VM in Azure, AWS, Google Cloud, or a VPS will do just fine for this purpose), and you'll likely want a domain name. If you've never worked with servers or DNS before, I'd recommend reading a bit about [DNS](https://www.redhat.com/en/blog/dns-domain-name-servers) and follow a [guide to setup a basic website](https://krystal.io/blog/post/beginner-s-guide-to-setting-up-a-vps-virtual-private-server).

# Part 1: Getting your platform up

Generally, in the year of our lord 2025, it's relatively easy to get a decent self hosted CTF platform up and running. For most people that's CTFd or rCTF. We'll focus on CTFd, because that's by far the most popular and very easy to get a basic set of containers running. 

To start, go to the [github repo](https://github.com/CTFd/CTFd) and clone the most recent stable version. I'd recommend cloning this into a directory with a name of the CTF you're running, because docker-compose will automatically namespace all containers by name of their containing directory (so if you always keep CTFd, they will all get namespaced together). That is:

```bash
mkdir my_ctf
git clone https://github.com/CTFd/CTFd.git my_ctf
```

This next step is optional, but is very useful to know how to do. You'll want to use your editor of choice to change the `docker-compose.yml` file to change the port bindings. The file will look something like the following:

```yaml
services:
  ctfd:
    build: .
    user: root
    restart: always
    ports:
      - "1337:8000" # change this line right here
    environment:
      - UPLOAD_FOLDER=/var/uploads
      - DATABASE_URL=mysql+pymysql://ctfd:ctfd@db/ctfd
      - REDIS_URL=redis://cache:6379
      - WORKERS=1
      - LOG_FOLDER=/var/log/CTFd
      - ACCESS_LOG=-
# file continues....
```
In this case, I've changed the port from 8000 to 1337 (by default the line looks like `"8000:8000"`) 

Next, you'll need to go ahead and build and run the containers, this is simple enough.

```bash
cd my_ctf
sudo docker compose build
sudo docker compose up -d
```

Now, your CTFd instance should be available at the port entered. You can test this by running `curl localhost:1337` (or whatever port you're binding to). By default this will run as HTTP only. It's 2025, TLSv1.3 is the standard, so how do we get that glorius lock icon? There are many ways of doing this, and if you already are using a reverse proxy like caddy, nginx, or something similar, you may already have a solution to this. If you don't have one already, I'd recommend caddy. It's very simple to use and will do most of the work for you.

To setup caddy for this, you'll want to go into a new directory, and create 2 files: another `docker-compose.yml` and a `Caddyfile`.

Put the following in your docker-compose.yml:
```yaml
version: '3'
services:
  caddy:
    container_name: caddy
    image: caddy
    restart: always
    ports:
      - '80:80'
      - '443:443'
    volumes:
      - ./Caddyfile:/etc/caddy/Caddyfile
```

Then the following in your Caddyfile:
```caddy
# Note: You'll need to change this to your actual domain name
my_ctf.example.com {
	# Note: You may need to change this port to whatever port you setup your CTFd instance to
	reverse_proxy localhost:1337
}
```

Now you can bring the container up, and it should automatically grab a certificate and handle proxying for you!

```bash
sudo docker compose up -d
```

Now that you have this set up, I'd recommend looking through the getting started guide for your chosen platform ([such as this for CTFd](https://docs.ctfd.io/tutorials/getting-started/)) and begin setting it up!

# Part 2: Challenge hosting

For many challenges, especially for categories like rev, misc, and often crypto, no additional hosting is necessary. In these cases, it's sufficient to simply add the challenges to your platform through the admin menu. 

When hosting is required however, it will generally fall under one of 3 categories. The first is to simply put it into a single container for everyone to access. This works well when there's no (or very little) global state, and it's difficult to modify the challenge as a standard user, this is often the case for simpler web challenges and some scripting based challenges. The second is jailed challenges. In jailed challenges, individual TCP connections are given new instances of a jailed binary --- this is very common for pwn based challenges. The last is fully instanced challenges. This is most commonly seen in more complex web challenges, but is seen around in other places as well. 

Note that this is non-exhaustive, but covers the vast majority of challanges. Notably one thing this doesn't cover is kernel pwn challenges, which require nested virtualization and some fairly complex infrastructure that I won't go into here.

## Part 2.0: General container hosting

These can range from very simple to very complex. For the most part, this is just about dockerizing your application, and hosting it like you would anything else. There are plenty of guides online for getting started with docker, but I'll link to [this one](https://wiki.tardisproject.uk/howto:specific:learndocker).

Depending on the scale of what you're hosting and for how long, you may consider anything from simple docker-compose files to kubernetes, but I'd recommend keeping it simpler with docker-compose to get started. You'd be surprised how far these can scale. 

## Part 2.1: Jailed binary hosting (mostly pwn)

For pwn challenges, there's actually been a relatively large amount of development into hosting them easily. I'd especially recommend using [redjail](https://github.com/redpwn/jail) for this purpose.

To start, I'll assume that you have some challenge and a Makefile to compile that challenge. If you don't have a Makefile, here's [a getting started guide](https://makefiletutorial.com/). From here, you'll want to create a Dockerfile in the directory of your challenge like so,
```Dockerfile
FROM gcc:13.3.0@sha256:845328b7cbdbd3cabd015eac5256043aebbea0f9081e74d479d3a2cdd840e6b5 AS builder
WORKDIR /builder/
COPY main.c Makefile ./ # Here you're just copying all your challenge files into the builder
RUN make

FROM debian:12.10-slim@sha256:b1211f6d19afd012477bd34fdcabb6b663d680e0f4b0537da6e6b0fd057a3ec3 AS bootstrap
WORKDIR /app/
COPY --from=builder /builder/buffer_overflow_0 ./run # Here you're copying all your compiled binaries from the builder to the challenge

FROM pwn.red/jail:0.4.1@sha256:ee52ad5fd6cfed7fd8ea30b09792a6656045dd015f9bef4edbbfa2c6e672c28c
COPY --from=bootstrap / /srv
COPY flag.txt /srv/app/ # This copies flag.txt into the served files -- you can also pass this through an environment variable if you prefer
```
With a Dockerfile like this, you're able to build and run the container with the following,
```bash
docker build -t my_chall .
sudo docker run -p 5001:5000 --priviledged my_chall
```
Note that the container must run as priviledged (the jail relies on having low level access to work), and you're binding to port 5000 in the container. From here, you'll distribute the challenge with a note to connect at your server and the port bound to, oftentimes in the format `nc <your_server> <your_port>`. In CTFd, you can put this in the connection information field.

From here, you'll need to host these in your container hosting platform of choice, likely integrate this into a docker-compose file.

## Part 2.2: Full instancing

This part is a work in progress, because web instancing is absolutely terrible. The general idea here is that every team gets the ability to create containers for challenges at will (usually with restrictions on quantity and how long they can leave them up). Common solutions involve a custom CTFd plugin integrating with a kubernetes cluster.

If you're considering doing this for a single challenge, it's usually more worth your time to adapt the challenge to not require full instancing, though I understand that's not a great answer.

