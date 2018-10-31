Notes on Docker:

==================================
Basics
==================================
Linux kernel has a colletion of namespaces - Mount (mnt), Process ID (pid), Network (net), Interprocess Communication (ipc), UTS, , User ID (user), Control group (cgroup)

Docker leverages Linux (and Windows) Names spaces and Cotrol Group. Each container gets its own islated set of namespaces and control groups.
Namespaces :
  PID           <-- each container gets its own process tree
  User          <-- each container gets its capability to define users and map to host users
  Net           <-- each container is able to define it's own NICs, IPs Routing Tables etc
  Mnt           <-- container gets isolated root file system
  IPC           <-- each container gets access to it's own shared memory
  UTS           <-- each container gets own hostname
  Control Groups - allow containers to allocate CPU, Memory, Disk IO
  
  These combined with Union File Systems (UFS allows files and directories of separate file systems, known as branches, to be transparently overlaid, forming a single coherent file system) are what allow containers to work as they do.
  

==================================
First steps
==================================

Running docker as non root:
      groupadd docker
      usermod -aG docker <docker user>
      chown root:docker /var/run/docker.sock
      systemctl stop docker
      systemctl start docker
To run docker as non-root - create a docker group (on Centos- on Ubuntu this is already done), then add your docker user to the group. Next change the owner of the docker socket to group "docker". Restart the docker engine.
  

To run with docker listening on the network rather than the internal docker socket:
   Ubuntu:
    docker -H 192.168.209.101:2375 -d &
   Centos
    nohup dockerd -H 192.168.209.101:2375  &
 Note you might need to open some firewall ports:
  firewall-cmd --add-port=237/tcp --permanent
  firewall-cmd --add-port=2375/tcp --permanent
  firewall-cmd --add-port=2376/tcp --permanent
  firewall-cmd --add-port=2377/tcp --permanent
  firewall-cmd --add-port=7946/tcp --permanent
  firewall-cmd --add-port=7946/udp --permanent
  firewall-cmd --add-port=4789/udp --permanent
On the docker client you then need to tell it the IP address and port to use:
  export DOCKER_HOST="tcp://192.168.209.101:2375"     <-- where 192.168.209.101:2375 are the IP:port of the docker host 
For the client to start listening to internal engine - just cleear thos variable:
  export DOCKER_HOST=
  
  
==================================
Containers and Images
==================================
  
Docker images - a collection of layers with a json manifest file which describes the image.
  docker pull redis                                   <-- unless otherwise specified, docker assume it should pull from dockerhub
    docker image pull redis
    Using default tag: latest
    Trying to pull repository docker.io/library/redis ... 
    latest: Pulling from docker.io/library/redis
    f17d81b4b692: Pull complete                         <-- each "pull complete" corresponds to a layer in the image
    b32474098757: Pull complete 
    8980cabe8bc2: Pull complete 
    2719bdbf9516: Pull complete 
    f306130d78e3: Pull complete 
    3e09204f8155: Pull complete

The manifest includes various info including a sha256 has of each layer content (hashing introduced in Docket 1.10 to increase security ie to ensure recived image==requested image. We can see the hash using:
      docker image ls --digests
REPOSITORY                                     TAG                 DIGEST                IMAGE ID            CREATED             SIZE
docker.io/ubuntu                               latest              sha256:29934af957c53004d7fb6340139880d23fb1952505a15d69a03af0d1418878cb   ea4c82dcd15a                                                                                             12 days ago         85.8 MB
docker.io/redis                                latest              sha256:481678b4b5ea1cb4e8d38ed6677b2da9b9e057cf7e1b6c988ba96651c6f6eff3   1babb1dde7e1                                                                                             12 days ago         94.9 MB
docker.io/centos                               latest              sha256:67dad89757a55bfdfabec8abd0e22f8c7c12a1856514726470228063ed86593b   75835a67d134                                                                                             3 weeks ago         200 MB





To spin up a centos container in interactive mode with tty terminal running bash:
  docker run -it centos /bin/bash
This opens container on command line with conatainer name in prompt:
  root@ce7c59657fda /]#
 
 From here you can perform linux tasks like cat, yum install (etc)
  cat /etc/hosts
  127.0.0.1	localhost
  ::1	localhost ip6-localhost ip6-loopback
  fe00::0	ip6-localnet
  ff00::0	ip6-mcastprefix
  ff02::1	ip6-allnodes
  ff02::2	ip6-allrouters
  172.17.0.2	ce7c59657fda
Note - container name is in hosts file

If we edit a file in the container, then kill the container, then start it back up the edit file is retained:
  yum install vi
  vi /tmp/tempfile
    "this is a test file"
  exit                                      <-- stop the container
Now check the container is stopped:
  docker ps
    CONTAINER ID        IMAGE     COMMAND                  CREATED             STATUS              PORTS               NAMES

  docker ps -a
    CONTAINER ID        IMAGE     COMMAND                  CREATED             STATUS              PORTS               NAMES
    ce7c59657fda        centos "/bin/bash"              19 hours ago        Exited (0) 2 minutes ago                   hopeful_leavitt

So container ce7c59657fda was exited 2 mins ago.
We can look inside the stopped containers file system:

(On Ubuntu with aufs file system this would be:
  ls -l /var/lib/docker/aufs/diff/ce7c59657fdasomerandomnumbers/tmp
  cat /var/lib/docker/aufs/diff/ce7c59657fdasomerandomnumbers/tmp/testfile
    "this is a test file"
 )
On Centos file overlay2 file system:
  ls -l /var/lib/docker/overlay2/85828d826b750a6a1e295876289c3f9ae6e0da3efb1004b28057858ccd94ae1b/diff/tmp
    -rw-r--r--. 1 root root 19 Oct 30 10:16 testfile
  cat /var/lib/docker/overlay2/85828d826b750a6a1e295876289c3f9ae6e0da3efb1004b28057858ccd94ae1b/diff/tmp/testfile
    this is a testfile
 
We can restart this container and attach to it:
  docker start ce7c59657fda
  docker attach ce7c59657fda

From the container command line we can then show that this file has persisted:
  [root@ce7c59657fda /]# cat /tmp/testfile
    this is a testfile


Lets dig in to some of the details of the docker run command:
  docker run -it fedora /bin/bash                           <-- we're firing up an interactive container based on fedora OS with bash CL
  Unable to find image 'fedora:latest' locally              <-- no fedora image found locally - so docker will search repository
  Trying to pull repository docker.io/library/fedora ...    <-- no private repo was specified so defaults to dockerhub
  latest: Pulling from docker.io/library/fedora             <-- pulling from dockerhub
  565884f490d9: Pull complete                               <-- pull complete
  Digest: sha256:b41cd083421dd7aa46d619e958b75a026a5d5733f08f14ba6d53943d6106ea6d
  Status: Downloaded newer image for docker.io/fedora:latest  <-- no image version was specified so "latest" was downloaded

We can pull images prior to running them for efficiency sake - eg overnight, pull them to local so they are pre-cached before running them

Eg to pull all fedora images from dockerhub:
  docker pull -a fedora
We can then list these :
  docker images fedora
    REPOSITORY          TAG                 IMAGE ID            CREATED             SIZE
    docker.io/fedora    29                  24508ec0e667        7 weeks ago         260 MB
    docker.io/fedora    28                  c582c1438f27        7 weeks ago         254 MB
    docker.io/fedora    latest              c582c1438f27        7 weeks ago         254 MB
    docker.io/fedora    rawhide             415235e73966        7 weeks ago         260 MB
    docker.io/fedora    26                  bb61c67169b5        7 weeks ago         232 MB
    docker.io/fedora    27                  7a2e85963474        7 weeks ago         236 MB
    docker.io/fedora    branched            30190780b56e        7 months ago        249 MB
    docker.io/fedora    25                  9cffd21a45e3        11 months ago       232 MB
    docker.io/fedora    26-modular          4f8db653ea36        11 months ago       85.3 MB
    docker.io/fedora    modular             4f8db653ea36        11 months ago       85.3 MB
    docker.io/fedora    24                  971e0f0a8b71        13 months ago       204 MB
    docker.io/fedora    20                  ba74bddb630e        2 years ago         291 MB
    docker.io/fedora    heisenbug           ba74bddb630e        2 years ago         291 MB
    docker.io/fedora    21                  ba6369d667d1        2 years ago         241 MB
    docker.io/fedora    22                  01a9fe974dba        2 years ago         189 MB
    docker.io/fedora    23                  60ba3309bebb        2 years ago         214 MB

Note image c582c1438f27 is tagged "latest" and also "28". Also, image 4f8db653ea36 is tagged 26-modular and modular, ba74bddb630e is tagged 20 and heisenbug etc - so images can have more than one tag

Images are stored on docker host under /var/lib/docker/<storage driver> eg for centos:
    ls /var/lib/docker/overlay2 
      0b240d3962d869e471de646d19b0b052ae1777d62cc006ce4b8a2679a512a6c1
      0f8deeca3e3f90a30fabbd03bd178315c3e0dc367ec5fec51e731b0d90c42339
      1dbe6801c09b2d5392130297a1fb692b962493644e4076a2b9451b9dbf296ffa
      21e8cce81d8476d37c3c0972f2f0f301f674c5593bc0da84812eae700e005aa2
      22a73683882664c8e42fed5169d1229d0ac5401f8e623a515173d838b82574cf
      22e5335684f051627b775da1e714edb79cf6a9b25a39e6541ad45aad8f2b2183
      etc...
  

Note docker images are "cut down" linux installs:
  docker images
  REPOSITORY                                     TAG                 IMAGE ID            CREATED             SIZE
  docker.io/ubuntu                               latest              ea4c82dcd15a        11 days ago         85.8 MB
  docker.io/centos                               latest              75835a67d134        2 weeks ago         200 MB
  docker.io/alpine                               latest              196d12cf6ab1        6 weeks ago         4.41 MB
  docker.io/hello-world                          latest              4ab4c602aa5e        7 weeks ago         1.84 kB
  docker.io/nigelpoulton/pluralsight-docker-ci   latest              07e574331ce3        3 years ago         557 MB

eg sizes above are 85.8Mb, 200Mb, 4.41 Mb etc

ets talk about containers
Images provide the cookie cutter - containers are the cookie:
Start the interactive container from the fedora image again:
  docker run -it fedora /bin/bash
To exit an interactive container without killing it :
  ctrl + P + Q
that leaves it running

We can see it running:
  docker ps
    CONTAINER ID        IMAGE               COMMAND             CREATED             STATUS              PORTS               NAMES
    5ec2626c154f        fedora              "/bin/bash"         2 minutes ago       Up 2 minutes                         vigilant_einstein

We can also see current and recently stopped containers:
  docker ps -a
    CONTAINER ID        IMAGE               COMMAND             CREATED             STATUS              PORTS               NAMES
    5ec2626c154f        fedora              "/bin/bash"         2 minutes ago       Up 2 minutes                        vigilant_einstein
    b34ff54638d0        fedora              "/bin/bash"         45 minutes ago      Exited (127) 28 minutes ago         heuristic_goldberg
    ce7c59657fda        centos              "/bin/bash"         21 hours ago        Exited (0) About an hour ago        hopeful_leavitt

Note - if a name isn't specified for the container then docker assigns one eg vigilant_einstein



Docker images are made up of multiple "layer images" or layers to avoid confusion. eg:
  docker pull coreos/apache
  Using default tag: latest
  Trying to pull repository docker.io/coreos/apache ... 
  latest: Pulling from docker.io/coreos/apache
  a3ed95caeb02: Pull complete                                                                         <-- layer
  5e160ca0bb5a: Downloading [======>                                            ] 8.031 MB/66.42 MB   <-- layer
  1f92e2761bfd: Downloading [=====================>                             ] 11.37 MB/26.17 MB   <-- layer
  1f92e2761bfd: Download complete

Previously these could be viewed by:
  docker images --tree
but this command was dropped in Docker version 1.6. as layer were re-engineered.
The tree view can still be shown using a custom tool from dockerhub:

  # if docker client using local unix socket
  alias dockviz="docker run -it --rm -v /var/run/docker.sock:/var/run/docker.sock nate/dockviz"

  # if docker client using tcp
  alias dockviz="docker run -it --rm -e DOCKER_HOST='tcp://127.0.0.1:2375' nate/dockviz"

Visualize the tree images by running:
  dockviz images -t

The way images were stored in dockerhub changed in Docker v1.1 to introduce better security. Layers are now hashed so that their contents cant be tampered with between a pull and subsequent push. This is explained here:
  https://windsock.io/explaining-docker-image-ids/
Since Docker v1.10, images and layers are no longer synonymous. Instead, an image directly references one or more layers that eventually contribute to a derived container's filesystem.
Layers are now identified by a digest (SHA256) 

  

We can see the different layers by listing /var/lib/docker/<storage driver>/layers eg:
  ls /var/lib/docker/overlay2           <-- centos
or
  ls /var/lib/docker/aufs/layers        <-- ubuntu

each entry listed is a file which holds a reference to the next "lower" layer in the stack eg:
  
a3ed95caeb02: Pull complete 
5e160ca0bb5a: Pull complete 
1f92e2761bfd: Pull complete 

The file systems created for each image can be seen :
Centos:
  ls -lar /var/lib/docker/overlay2/<layer>/diff/
    drwxrwxrwt. 2 root root 20 Oct 30 14:45 tmp
    drwxr-xr-x. 3 root root 21 Oct 30 11:45 run
    dr-xr-x---. 2 root root 27 Oct 30 14:45 root
    drwx------. 5 root root 69 Oct 30 11:45 ..
    drwxr-xr-x. 5 root root 40 Oct 30 11:45 .

Lets create a new image named "fridge" based on a container we create from an ubuntu image with some added content:
  docker run ubuntu /bin/bash -c "echo 'cool content' >/tmp/coolfile"     <-- short lived container that echos content into a file
  docker ps -a                                                            <-- find our container ID
  docker commit <container-id> fridge                                     <-- same the container as an image
  docker images
    REPOSITORY                                     TAG                 IMAGE ID            CREATED             SIZE
    fridge                                         latest              f6adca93f8bb        8 seconds ago       85.8 MB
    docker.io/ubuntu                               latest              ea4c82dcd15a        11 days ago         85.8 MB
    docker.io/centos                               latest              75835a67d134        2 weeks ago         200 MB
    docker.io/alpine                               latest              196d12cf6ab1        6 weeks ago         4.41 MB
  
  docker history
    IMAGE               CREATED             CREATED BY                                      SIZE                COMMENT
  f6adca93f8bb        21 seconds ago      /bin/bash -c echo 'cool content' > /tmp/co...   13 B                
  ea4c82dcd15a        11 days ago         /bin/sh -c #(nop)  CMD ["/bin/bash"]            0 B                 
  <missing>           11 days ago         /bin/sh -c mkdir -p /run/systemd && echo '...   7 B                 
  <missing>           11 days ago         /bin/sh -c rm -rf /var/lib/apt/lists/*          0 B                 
  <missing>           11 days ago         /bin/sh -c set -xe   && echo '#!/bin/sh' >...   745 B               
  <missing>           11 days ago         /bin/sh -c #(nop) ADD file:bcd068f67af2788...   85.8 MB 

The <missing> value in the IMAGE field for all but one of the layers of the image, is misleading. It incorrectly suggests an error, because since Docker v1.1 layers are no longer synonymous with a corresponding image (see https://windsock.io/explaining-docker-image-ids/).
  
The dockviz tree util shows that fridge is tagged as  "latest" and based on ubuntu (latest) image:
  dockviz images -t
  ├─<missing> Virtual Size: 85.8 MB
  │ └─<missing> Virtual Size: 85.8 MB
  │   └─<missing> Virtual Size: 85.8 MB
  │     └─<missing> Virtual Size: 85.8 MB
  │       └─ea4c82dcd15a Virtual Size: 85.8 MB Tags: docker.io/ubuntu:latest
  │         └─f6adca93f8bb Virtual Size: 85.8 MB Tags: fridge:latest
  
  
Before repositories were widely in use, images were moved from host to host using docker save and docker load:
We can save our docker image - eg save fridge image to a tar file in /tmp:
  <host1>$ docker save -o /tmp/fridge.tar fridge
We could then copy this to another host and load it into the docker engine using docker load:
  <host2>$ docker load -i /tmp/fridges.tar
  <host2>$ docker run -it fridge /bin/bash

Every layer gets a thin writeable to layer which is where all changes to the container/image (if committed) go. 
All other layers aside from the top layer are read only. All changes are in the top layer. 
Union file systems give containers the appearance of a "normal" linux system even though this is happening under the covers to to apps (and users) the container appears like any other linux container

Lets run a container that pings the google name server 30 times:
  docker run -d ubuntu /bin/bash -c "ping 8.8.8.8 -c 30"
we can see this running:
  docker ps
  CONTAINER ID        IMAGE               COMMAND                  CREATED             STATUS              PORTS               NAMES
  f19009357e26        ubuntu_ping         "/bin/bash -c 'pin..."   4 seconds ago       Up 3 seconds                           goofy_bhabha

We can view details of the running processes INSIDE THE CONTAINER using docker top:

  docker top f19009357e26
  UID     PID                 PPID                C                   STIME               TTY       TIME             CMD
  root    59536               59521               0                   16:17               ?         00:00:00         ping 8.8.8.8 -c 300
    

Docker run command:
docker run has any options eg:
  docker run -it                        <-- run interactive with command line
  docker run -d                         <-- run in background (detached)
  docker run --cpu-shares=256           <-- controls CPU shares - 256=quarter, 1024=all.By default all conatiners on host get equal access
  docker run --memory=1g                <-- container has access to 1Gb memory 
  ... lots more options - read the man for details
  
Each container "usually" is set up to have a single process running inside it - 
Lets start a container in background running a continuous ping:
  docker run -d ubuntu:14.04.1 /bin/bash -c "ping 8.8.8.8"
we can see it running:
  docker ps
we can see lots of detail about the container and its process:
  docker inspect 95447ecd5d6a
  [
     {
         "Id": "95447ecd5d6a923f2b33a02c74bfa06ed734a83e27e61170ae1c7ca46b206b77",
         "Created": "2018-10-30T16:32:22.638009435Z",
         "Path": "/bin/bash",
         "Args": [
             "-c",
             "ping 8.8.8.8"
         ],
         "State": {
             "Status": "running",
  ....
  
We can attach to the running process:
  docker attach 95447ecd5d6a
    64 bytes from 8.8.8.8: icmp_seq=114 ttl=127 time=18.8 ms
  64 bytes from 8.8.8.8: icmp_seq=115 ttl=127 time=18.6 ms
  64 bytes from 8.8.8.8: icmp_seq=116 ttl=127 time=30.4 ms
  64 bytes from 8.8.8.8: icmp_seq=117 ttl=127 time=18.4 ms
  64 bytes from 8.8.8.8: icmp_seq=118 ttl=127 time=54.2 ms
  64 bytes from 8.8.8.8: icmp_seq=119 ttl=127 time=14.1 ms

We can quit the process with ctrl c but this symultaneously kills it:
  ctrl + C          <-- quit and kill

We can tail the container logs using:
  docker container logs -f <container id>


==================================
Containers Management
==================================
Start an interactive container
  docker run -it ubuntu:14.04.1 /bin/bash
quit the container without killing it
  crtl+P+Q
Re-attach to the running container:
  docker attach <container ID>
Stop the container:
    docker stop <container id>
Kill the container
    docker kill <container id>
Start a container:
    docker start <container id>
Restart a container:
    docker restart <container id>

PID 1 and containers:
In the linux world, PID1 is usually the "init" which is the mother of all processes - manageing and clearing up when something goes wrong.
Docker containers run processes as PID1 - but it isn't a "proper" PID1 in the linux sense - it doesn't handle kill signals etc.
So running more than one process in a container can be done but with care - and the best practice is 1 container, 1 process

We can see how many containers and images we have using:
  docker info
    Containers: 18
      Running: 1
      Paused: 0
      Stopped: 17
    Images: 11
      .... etc
We can remove a non-running container using :
  docker rm <container id>
We can remove a container (non-running) using :
  docker rm -f <container id> 
or by stopping it first:
  docker stop <container id>
  docker rm <container id>
  
  

To get a command line on a running container (ssh is frowned on as containers should be lightweight and secure) but there are multiple methods for doing this:
1) We can use "nsenter":
This is a utiltiy which allows users to "enter a namespace" - nsenter requires the container pid:
  docker ps
CONTAINER ID        IMAGE               COMMAND             CREATED             STATUS              PORTS               NAMES
c00398f8bd1e        ubuntu:14.04.1      "/bin/bash"         32 minutes ago      Up 32 minutes                           elated_jepsen

  docker inspect c00398f8bd1e| grep -i pid
    "Pid": 60404,
 The command fo nsenter is:
  nsenter -m -u -n -p -i -t <pid>
eg
  nsenter -m -u -n -p -i -t 60404 /bin/bash
or if not running as root:
  sudo   nsenter -m -u -n -p -i -t 60404 /bin/bash
and ctrl-c takes the user back to the host CL without killing the container
2) We can use "docker exec":
    docker exec -it <container-id> /bin/bash 
"exit" takes the user back to the host CL without killing the container


  










# Simple web app for Pluralsight courses and Docker Deep Dive book
This is a quick and dirty node.js app cobbled together for the purposes of demonstrating how to Dockerize/containerize a simple app.

**This app is not maintained and WILL be full of vulnerbilities. Use at own risk**

Exposes web server on port 8080 as per ./app.js

See **Dockerfile** for more details
