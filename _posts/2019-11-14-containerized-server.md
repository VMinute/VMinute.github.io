# Build a "containerized" home server

I spent the last months working on a new solution for embedded devices named [Torizon](https://www.toradex.com/operating-systems/torizon).  
One of the main features of Torizon is his capability to run docker containers on an embedded device. This is pretty new for devices, but containers are already widely used on servers and in the cloud. Working with containers I appreciated the capability of packing an application and all his user-mode dependency in an easy to use format and also the fact that running containers has much less overhead than running a full-blown virtual machine.  
I have a linux-based home server at home, providing service like file-sharing, printer-sharing, media-library etc. but its hardware is quite old and showing some issues (I have to power-cycle it more or less once per week). Also, maintaining his main OS (Ubuntu 16.04) has become quite challenging, fighting with low disk space, broken installs, missing dependencies etc.  
I decided that replacing my home server is a good chance to use containers in a practical scenario. I know that this will probably need more storage for the different image and to find some "creative" ways to install packages and keep their configuration and data on the host file system to make backups and updates easier to manage.  
The plan is also to not store any permanent information inside the containers, so it will be possible to rebuild and update them as needed, without having to restore or reconfigure them.  
I also need to improve my Linux skills and this project will give me a chance to do some extra practice in my free time.
Those are the services I plan to run on my home server:

- CUPS for printer sharing
- Samba to share folders with pictures, music, documents and personal folders for me, my wife and the kids
- PLEX server to share media and access them from TV and other devices
- backup server to safely archieve data from other PCs (and macs) in the house
- git server to manage code of my personal projects
- whatever comes to my mind and I want to experiment with :) 

Some services may be designed to run inside a container, other may not, I'll try to configure them in the simplest possible way, sharing resources from the host when needed but trying to stick to some "principles":

- Containers should be updated by rebuilding them, this means that all configuration and data must be stored outside of the container itself;
- number of running processes inside a container should be kept to the minimum, avoiding duplication between containers and the host os;
- containers should use a user account when running and not root, users inside the container will match local users on host OS (same UID) if needed;
- each container it's going to be separated from the others, unless multi-container solutions are needed;
- service could be stopped/restarted by stopping starting the container, this may involve some scripts, at least to build command line;
- for containers that will need components built from source I'll use multi-stage builds in docker, generating a smaller target container;
- each service will have a two folders, one for building it, one for running it, building will involve for sure a Dockerfile, but may involve also additional resources, if possible data and configuration information will be kept separate for runtime;
- all these principles can be violated if I have a good reason to do that :) and this will be documented;

## Hardware

I try to keep my budget under 300€, re-using some parts from the old server.  
I don't need a lot of processing power and I wanted to use a fanless CPU solution to reduce noise (even if I have fans in case and power supply).  
I work on ARM-based devices but I think that for "beefy" servers a x64 machine still has some advantages and ARM64 hardware with similar specs would be more expensive and not easy to find on the consumer market. But using containers will help me to move to a different CPU architecture in the future, I hope.  
On the other side I will need plenty of storage, mixing SSDs (for OS and applications) and HDDs for storing large amounts of data.  
I started by mounting the motherboard, PSU and SSDs I plan to use for the OS on my desk, keeping the old server running in the meantime. When I'll have most of the things working I'll move the stuff inside the case and connect the HDDs.
Here's a list of the components:  

| type        | component                     | price (€) | notes |
|-------------|-------------------------------|-----------|-------|
| motherboard | Asrock J3455-ITX              |       88  | I got it because it's the same chipset used for mini-linux PCs and I didn't want compatibility issues |
| memory      | Crucial CT2K102464BF186D 16GB |       107 | Using containers requires more RAM than running regular applications because you will not leverage shared libraries, 16GB seems a good amount of memory for an home server |
| PSU         | 500W STX PSU                  |        25 | |
| SSD         | Sandisk 240GB                 |        38 | (from an old PC) Used to store OS |
| SSD         | Sandisk 480GB                 |        61 | Used to store data and home folders |
| HDDs        | Seagate 2TB x 2               |      used | Storage from the old server |
| USB HDD     | 2TB                           |      used | Used for local backups |
| case        | mini-itx case                 |      used | case from old server, easy accessible bays for HDDs, not too noisy |


## Main OS

Since I don't plan to run any of the services directly from my host OS I decided to use Alpine Linux as my main OS. Alpine is widely used as base-image for containers because it has a very low footprint. I wanted to use it on my server to keep the base OS to the bare minimum, leaving storage and RAM for the containers.  
I istalled their standard x64 image and added a few packages:

- zsh (Alpine uses ash, personal preferences)
- wget curl git (to download code and binaries from the command line)
- util-linux (adding some useful tools)
- sudo (I don't like to be logged-in as root)
- docker (should I explain why?)
- python (my favorite language for scripring)
- man (to have reference for the commands ready from the shell)
- tree (useful to show how I organized folders)

### main os configuration

Those are some of the changes I made to the default configuration. Some are needed to operated the server in the way it's intended to operate, other are just personal preferences related to the tools I like to use.  

#### Starting docker at boot

This is required to have your system up and running without any need to execute commands interactively.
It's pretty easy to enable docker at boot in Alpine, just type (as root);

```zsh
rc-update add docker boot
```

### Sudo

As I said, I don't like to be logged in as root, is too easy to make devastating mistakes. I prefer to use sudo to run single commands as root only when needed (this may also lead to disasters, but should be a bit more complicated).
I plan to add to a group all users that will have "superpowers". On Alpine this group is named wheel and to enable it you have to run visudo and uncomment the line enabling it

```zsh
%wheel ALL=(ALL) ALL
```

Sudo will always ask for a password, this is, again, an extra measure to avoid doing stupid things as root.

### Users and groups

I want to keep the "maintenance" of containers separate from my personal account on the server, so I will create a user named "dockeruser" that will be used to build and manage containers, keeping all required data inside its own home folder.  
I use UID 2000 because I plan to user UID range 1000-1999 for users that are "shared" between local system and containers.  
I also add the user to the groups that will allow him to use docker and invoke root superpowers when needed.

```zsh
adduser -h /home/dockeruser -s /bin/bash -u 2000 dockeruser
addgroup dockeruser docker
addgroup dockeruser wheel
```
Alpine enables users to login via ssh by default, so now I'll be able to operate on my server from other PCs in my local network.

### folders

In the dockeruser home folder I plan to store scripts, configuration files and additional files required for each container.  
I'll also keep some documentation about the system and, specifically, the ports used by each container (useful to solve conflicts)

## First container!

The first application I plan to run is [portainer](www.portainer.io).  
It's an application used to monitor...containers, so it will have much to do on my system at the moment, but it will be useful to check that docker, networking and the rest of the system work.  
Portainer is already designed to be distributed and executed as a container, so no work is needed to "containerize" it.  
I anyway want to keep it's data stored permanently out of the container, so I need to share a local folder.  
Under dockeruser home I created a folder for containers and one for docs. Inside container folders I plan to have subfolders for build, configuration and data.  
Portainer doesn't need to be built and stores configuration and data in the same folder, so this makes the folder structure quite simple.

```zsh
.
├── containers
└── docs
    └── ports
```

First we need to create a user that will be used to run portainer.
We also need to create folders under containers to store application data (portainer keeps configuration and data in the same folder)

```zsh
sudo adduser -D -H portainer
mkdir -p containers/portainer/data
sudo chown portainer containers/portainer/data
```

The user will have to access docker socket (as the command line interface), so we need to get id of the docker group

```zsh
~$ cat /etc/group | grep "docker:"
docker:x:102:dockeruser
```
Time to run portainer as a container!

```zsh
~$ docker run -d --restart always -v $(pwd)/containers/portainer/data:/data -v /var/run/docker.sock:/var/run/docker.sock -p 9000:9000 --name portainer --user 1000:102 portainer/portainer
Unable to find image 'portainer/portainer:latest' locally
latest: Pulling from portainer/portainer
d1e017099d17: Pull complete 
0b1e707a06d2: Pull complete 
Digest: sha256:d6cc2c20c0af38d8d557ab994c419c799a10fe825e4aa57fea2e2e507a13747d
Status: Downloaded newer image for portainer/portainer:latest
ac72d16741bbef2a34fc8d4d3168ab79253b67af4ed32a358bb8fd74c19413d1
```

Connecting using a browser to my server IP address on port 9000 will show me status of my server and of the running containers.  
Obviously I can get the same information from the command line:

```zsh
~$ docker image ls
REPOSITORY            TAG                 IMAGE ID            CREATED             SIZE
portainer/portainer   latest              19d07168491a        6 weeks ago         74.1MB
~$ docker container ls
CONTAINER ID        IMAGE                 COMMAND             CREATED             STATUS              PORTS                    NAMES
ac72d16741bb        portainer/portainer   "/portainer"        45 seconds ago      Up 44 seconds       0.0.0.0:9000->9000/tcp   portainer
```

Some notes on the command line used to start portainer:

- I assigned a name to the instance I started. If you have containers that will run in a single instance (and that would be probably true for most of the things I'll run on my server) using a descriptive name will make them much easier to identify and manage. Docker generates funny names for new instances, but probably not as easy to remember as names you choose.
- Portainer uses port 9000, this must be explicitely enabled on the command line, ports that are exposed via docker file are not automatically exposed when you run the container.
- The app stores its data under /data, this folder has been mounted from the /containers/portainer/data subfolder. If you check contents you'll see that those belong to portainer user (uid=1000).
- To let portainer manage containers we should also mount the socket that docker exposes for its client applications. This is accessible for users of group docker (gid=102) that is the one we passed to the --user argument. We don't need to add local portainer user to the host OS group, /etc/group won't be accessible inside the container anyway.
- The --restart always option will ensure that the container will be restarted even after a system reboot.