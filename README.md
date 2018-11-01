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

(NB Containers need to be linux on linux or windows on windows)

Image layers work like:
  Base LAyer                    <-- Kernel file system (eg Centos) - this makes container look and feel like selected OS on top of host OS
  Layer 2 - eg App layer        <-- deployed application
  Layer 3 - eg updates          <--- any applied updates
  Layer 4 - writeabvle layer    <-- top layer is the only writeable layer
Union file system makes this look like a single coherent image

We can see references to our layers in /var/lib/docker/<file system driver>/diff  or /var/lib/docker/<file system driver>/ eg:
  ls /var/lib/docker/overlay2/
  ls /var/lib/docker/aufs/diff
  
 We can see how each layer gets created:
  docker history redis
  IMAGE               CREATED             CREATED BY                                      SIZE                COMMENT
  1babb1dde7e1        12 days ago         /bin/sh -c #(nop)  CMD ["redis-server"]         0 B                 
  <missing>           12 days ago         /bin/sh -c #(nop)  EXPOSE 6379/tcp              0 B                 
  <missing>           12 days ago         /bin/sh -c #(nop)  ENTRYPOINT ["docker-ent...   0 B                 
  <missing>           12 days ago         /bin/sh -c #(nop) COPY file:b63bb2d2b8d095...   374 B               
  <missing>           12 days ago         /bin/sh -c #(nop) WORKDIR /data                 0 B                 
  <missing>           12 days ago         /bin/sh -c #(nop)  VOLUME [/data]               0 B                 
  <missing>           12 days ago         /bin/sh -c mkdir /data && chown redis:redi...   0 B                 
  <missing>           12 days ago         /bin/sh -c set -ex;   buildDeps='   ca-cer...   36.3 MB             
  <missing>           12 days ago         /bin/sh -c #(nop)  ENV REDIS_DOWNLOAD_SHA=...   0 B                 
  <missing>           12 days ago         /bin/sh -c #(nop)  ENV REDIS_DOWNLOAD_URL=...   0 B                 
  <missing>           12 days ago         /bin/sh -c #(nop)  ENV REDIS_VERSION=5.0.0      0 B                 
  <missing>           2 weeks ago         /bin/sh -c set -ex;   fetchDeps="   ca-cer...   3 MB                
  <missing>           2 weeks ago         /bin/sh -c #(nop)  ENV GOSU_VERSION=1.10        0 B                 
  <missing>           2 weeks ago         /bin/sh -c groupadd -r redis && useradd -r...   329 kB              
  <missing>           2 weeks ago         /bin/sh -c #(nop)  CMD ["bash"]                 0 B                 
  <missing>           2 weeks ago         /bin/sh -c #(nop) ADD file:f8f26d117bc4a92...   55.3 MB 
Each non-0b SIZEd action results in a layer - so redis image will have 5 layers (exept in this case the mkdir action is so small it gets rounded to 0 - so redis has 6 layers). The actions which dont create layers are adding details to the image json file
    
We can see how many layers and their hashes in an image by running docker image inpect:
  docker image inspect redis
              .....
                "Layers": [
                "sha256:237472299760d6726d376385edd9e79c310fe91d794bc9870d038417d448c2d5",
                "sha256:197ffb073b01a6995ad4b0c78885a7ae78e0ff7aac93c1020ebaec8ad7a154fe",
                "sha256:aa1a19279a9ad6c1e94e98ad45c1d27a0432b8e90706da6ffdcef10586537131",
                "sha256:79a66d80dbb46cc128520a92d82fd55a8efc42f6a10b2974ab247f1de2a206e2",
                "sha256:edf7dde91cc5d207b2f1aa8c364f26409d5b3681f31a4a5a1832404e43fae74c",
                "sha256:b643e67918745678b550890c331addbe7364757b9b018441531e1d5c04f9a93c"
            ]
            .....
We can delete an image using "docker image rm" - which also removes corresponding files from /var/lib/docker
  sudo ls /var/lib/docker/overlay2|wc -l
  76

  docker image rm redis
  Untagged: redis:latest
  Untagged: docker.io/redis@sha256:481678b4b5ea1cb4e8d38ed6677b2da9b9e057cf7e1b6c988ba96651c6f6eff3
  Deleted: sha256:1babb1dde7e1fc7520ce56ce6d39843a074151bb192522b1988c65a067b15e96
  Deleted: sha256:68f3c8e2388da48dd310e4642814feca68081445635716be58d7ebb69b611922
  Deleted: sha256:b18dd54614f34239abc8a1165c90d5416a413d1f4c3c6711648e49e26e4445e7
  Deleted: sha256:bf9efae34b1e94384b8cd011cf71591efab734b57961017bad608be56b7b1c9c
  Deleted: sha256:7ae66985fd3a3a132fab51b4a43ed32fd14174179ad8c3041262670523a6104c
  Deleted: sha256:bf45690ef12cc54743675646a8e0bafe0394706b7f9ed1c9b11423bb5494665b
  Deleted: sha256:237472299760d6726d376385edd9e79c310fe91d794bc9870d038417d448c2d5

  sudo ls /var/lib/docker/overlay2|wc -l
  70

Registries:
Images are stored in repositories in registries. The official docker registry is dockerhub (amazon, google etc also have registries) but on-premise registries (eg docker's DTR Docker Trusted Registry) are also available

 NB -  the layers stored on the file-system have content-hashes. 
      Before storing in a registry the layers are compressed - therefor the store layers dont match the content hash. 
      The compressed layers therefore get a second hash - the distribution-hash
 
 

Containers:

Images are a collection of read-only layers.
A container is a read-write image running on top of an image. Many containers can share a single image so the image needs to be read-only.
All changes eg when an app/process needs to create a file or amend an existing file, are done in the writeable layer - if a file from a read-only layer is being amended then it gets copied to the writeable layer - "copy-on-write"


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

===
Note!!! the tag "latest" is a manually added tag - just because a repo is tagged as latest - it doesn't mean it is the latest version!!
===


Note docker images are "cut down" linux installs:
  docker images
  REPOSITORY                                     TAG                 IMAGE ID            CREATED             SIZE
  docker.io/ubuntu                               latest              ea4c82dcd15a        11 days ago         85.8 MB
  docker.io/centos                               latest              75835a67d134        2 weeks ago         200 MB
  docker.io/alpine                               latest              196d12cf6ab1        6 weeks ago         4.41 MB
  docker.io/hello-world                          latest              4ab4c602aa5e        7 weeks ago         1.84 kB
  docker.io/nigelpoulton/pluralsight-docker-ci   latest              07e574331ce3        3 years ago         557 MB

eg sizes above are 85.8Mb, 200Mb, 4.41 Mb etc

Lets talk about containers
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
but this command was dropped in Docker version 1.6. as layers were re-engineered to use hashing for security.

(((NB
The tree view can still be shown using a custom tool from dockerhub:

  # if docker client using local unix socket
  alias dockviz="docker run -it --rm -v /var/run/docker.sock:/var/run/docker.sock nate/dockviz"

  # if docker client using tcp
  alias dockviz="docker run -it --rm -e DOCKER_HOST='tcp://127.0.0.1:2375' nate/dockviz"

Visualize the tree images by running:
  dockviz images -t
  )))

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
Container Management
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

CMD and ENTRYPOINT:
Images can have a CMD or ENTRYPOINT (& CMD)
If a CMD only is specified then this becomes the default program the container will run on startup
If an ENTRYPOINT is defined this this becomes the default program the container will run on startup and any CMD string will be appended as args to the ENTRYPOINT.
Both values can be seen in the Dockerfile or viewed on an image or a container by:
  docker image inspect <image name>
  eg
  docker image inspect alpine
                ],
              "Cmd": [
                 "/bin/sh"
              ],
              "ArgsEscaped": true,
              "Image": "sha256:836dc9b148a584f3f42ac645c7d2bfa11423c1a07a054446314e11259f0e59b7",
              "Volumes": null,
              "WorkingDir": "",
              "Entrypoint": null,
  docker container inspect <container name>
  eg
  docker container inspect 5ab7bc3ac6bf
               ],
             "Cmd": null,
              "ArgsEscaped": true,
              "Image": "psweb",
              "Volumes": null,
              "WorkingDir": "/src",
              "Entrypoint": [
                 "node",
                 "./app.js"
             ],


Both of these can be specifically over-ridden from the command line on a "docker run" command
If command line args are passed to a "docker run" command then for an container with CMD only specied, the CLI args become the default process to run, for a container with an ENTRYPOIN defined the CLI Args become appended to the ENTRYPOINT

Docker recommends using ENTRYPOINT to set the image’s main command, and then using CMD as the default flags. Here is an example Dockerfile that uses both instructions. eg:
  
  FROM ubuntu
  ENTRYPOINT ["top", "-b"]
  CMD ["-c"]

 
 Container ports:
 
 Specifying EXPOSE in the Dockerfile exposes a port from the running container to the outside world
 This can be mapped to a port on the host using the -p flog:
  docker container run -d  --name web1 -p 80:80 nginx          <-- runs container based on nginx mapping host port 80 to container port 80

we can see port mappings using docker port:
  docker port web1
    80/tcp -> 0.0.0.0:80          <-- listening on all addresses on port 80
To cleanup
  docker container rm $(docker container ls -aq) -f     <-- force remove results of "docker list all"
    
===============================
Docker logging
===============================
2 types - daemon and container (or app) logs

Docker daemon logs:
To see daemon logs do either:
  journalctl -u docker.service
 or
  tail -f /var/log/messages

Docker container logs:
Docker "expects" logs get written to stdout and stderr - so ideally apps run as PID1 and write logs to these. If not then log files can be sym-linked or mounted in a volume container.
Since docker 17 enterprise docker has log drivers - plugins for splunk, syslog, gelf (etc) - these forward container logs to logging solutions.
Specify log drivers to use in the:
  daemon.json
Log drivers for specific containers can be overridden using:
  --log-driver --log-opts

Any logs created can then be inspected using:
  docker logs <container>



===============================
Docker swarm
===============================
Swarm central to docker going forward 
Swarm has 2 elements - secure clustring and orchestration. Orchestration may go to Kubernetes but secure clustering is key to docker future.

Swarm mode: since 1.12 swarm mode (clustering workload management etc) has been build in

Start swarm using:
  dockerswarm init
This docker host becomes the 1st swarm manager. It becomes:
  swarm leader
  swarm root CA (external CA's can be configured using swarm init --external-ca flag) so it can issue certs to the swarm members
  gets a client certificate
  creates an encrypted secure "etcd" cluster store which gets automatically distributed to any other managers in the swarm
  (etcd is a distributed key value store that provides a reliable way to store data across a cluster of machines. It gracefully handles     leader elections during network partitions and will tolerate machine failure, including the leader. Applications can read and write data   into etcd eg to store database connection details or feature flags in etcd as key value pairs. These values can be watched, allowing       your app to reconfigure itself when they change)
  Creates cryptographoc join tokens for new managers and for new workers

Multiple swarm managers can be created and the auto-sync but only one is ever active.
If a manager fails - another is elected to be the leader. This is all handled by RAFT - When the Docker Engine runs in swarm mode, manager nodes implement the Raft Consensus Algorithm to manage the global cluster state.
Best practices fpr managers - have 3, 5 or 7 - low numbers improve decision making, odd numbers improve quorum and avoid split-brain.

Also - raft works best on fast reliable networks so (for instance) dont place maagers in different amazon regions

Worker nodes dont get access to the cluster store but they get a cert issued by the manager that identifies the worker, it's swarm and it's role (worker) and they also get a list of IPs for all managers should any one manager fail. Workers do all the app work.


===============================
Building a swarm
===============================
First check if we are in swarm mode:
  docker system info
    ...
     Swarm: inactive
    ...

To create the swarm ( and therefore make this leader and root CA, issue a client cert, create secure encrypted distributed cluster store, create join tokens and cert rotation policy):
  docker swarm init
    Swarm initialized: current node (k99q6ic98c519uf2ofbe0s4jc) is now a manager.

    To add a worker to this swarm, run the following command:

    docker swarm join \
    --token SWMTKN-1-4m9zv4int1uglw5viwwxw8u4gr8nc3zumhbf9t2nncabr8w2w1-3w34mnkpastor6lgpb7bzsyy0 \
    192.168.209.101:2377

    To add a manager to this swarm, run 'docker swarm join-token manager' and follow the instructions.

docker system info now shows swarm mode is active along with details of the swarm :
  docker system info
    Swarm: active
    NodeID: k99q6ic98c519uf2ofbe0s4jc
    Is Manager: true
    ClusterID: wxaqlcr7s1umy9y8opp4ou0ub
    Managers: 1
    Nodes: 1
    Orchestration:
      Task History Retention Limit: 5
    Raft:
     Snapshot Interval: 10000
      Number of Old Snapshots to Retain: 0
      Heartbeat Tick: 1
      Election Tick: 3
    Dispatcher:
      Heartbeat Period: 5 seconds
    CA Configuration:
     Expiry Duration: 3 months
    Node Address: 192.168.209.101
    Manager Addresses:
     192.168.209.101:2377

We can also see details of the swarm using docker node:
  docker node ls
    ID                           HOSTNAME         STATUS  AVAILABILITY  MANAGER STATUS
    k99q6ic98c519uf2ofbe0s4jc *  centos1.gjj.com  Ready   Active        Leader

To add a new manager - first get the manager token:
  docker swarm join-token manager
    To add a manager to this swarm, run the following command:

    docker swarm join \
    --token SWMTKN-1-4m9zv4int1uglw5viwwxw8u4gr8nc3zumhbf9t2nncabr8w2w1-45ond6y53p1ci09w5pab6vnrp \
    192.168.209.101:2377

copy that string, switch to another docker host and paste and the second host will join the swarm as a manager:
  docker swarm join \
    --token SWMTKN-1-4m9zv4int1uglw5viwwxw8u4gr8nc3zumhbf9t2nncabr8w2w1-45ond6y53p1ci09w5pab6vnrp \
    192.168.209.101:2377
      This node joined a swarm as a manager.

Now docker node shows two nodes (one leader, one "reachable"):
  docker node ls
    ID                           HOSTNAME         STATUS  AVAILABILITY  MANAGER STATUS
    k99q6ic98c519uf2ofbe0s4jc    centos1.gjj.com  Ready   Active        Leader
    rk721raqkonplaivadjiw7pj8 *  centos2.gjj.com  Ready   Active        Reachable

Lets add a worker:
  docker swarm join-token worker
    To add a worker to this swarm, run the following command:

    docker swarm join \
    --token SWMTKN-1-4m9zv4int1uglw5viwwxw8u4gr8nc3zumhbf9t2nncabr8w2w1-3w34mnkpastor6lgpb7bzsyy0 \
    192.168.209.100:2377
switch to another host:
    docker swarm join \
    >     --token SWMTKN-1-4m9zv4int1uglw5viwwxw8u4gr8nc3zumhbf9t2nncabr8w2w1-3w34mnkpastor6lgpb7bzsyy0 \
    >      192.168.209.100:2377
    This node joined a swarm as a worker.
 
Workers cannot query the node store:
  docker node ls
    Error response from daemon: This node is not a swarm manager. Worker nodes can't be used to view or modify cluster state. Please run        this command on a manager node or promote the current node to a manager.

but on a manager node we now see this node added:
  docker node ls
    ID                           HOSTNAME         STATUS  AVAILABILITY  MANAGER STATUS
    2zzxj0qw7lumuol7kwinla5sn    centos4.gjj.com  Ready   Active        
    k99q6ic98c519uf2ofbe0s4jc    centos1.gjj.com  Ready   Active        Reachable
    rk721raqkonplaivadjiw7pj8 *  centos2.gjj.com  Ready   Active        Leader
The blank status shows it is a worker node

To rotate the client tokens issue (on a manager):
  docker swarm join-token --rotate worker
      Successfully rotated worker join token.

    To add a worker to this swarm, run the following command:

        docker swarm join \
       --token SWMTKN-1-4m9zv4int1uglw5viwwxw8u4gr8nc3zumhbf9t2nncabr8w2w1-dtpq3yr3otbs2eeo1oeudtzqb \
        192.168.209.100:2377

Existing node membership is unnaffected but the old join token will no longer work

To view the client cert:
  sudo openssl x509 -in /var/lib/docker/swarm/certificates/swarm-node.crt -text
we can see for example that the organisation is the swarm id, the organisational unit is the node's role and the canonical name is the cryptographic node id.
On a worker:
sudo openssl x509 -in /var/lib/docker/swarm/certificates/swarm-node.crt -text
  Subject: O=wxaqlcr7s1umy9y8opp4ou0ub, OU=swarm-worker, CN=2zzxj0qw7lumuol7kwinla5sn
or on a manager:
  Subject: O=wxaqlcr7s1umy9y8opp4ou0ub, OU=swarm-manager, CN=rk721raqkonplaivadjiw7pj8


=============================
Auto-locking a swarm
=============================
The Autolock feature affects managers only - not workers

In Docker 1.13 and higher, the Raft logs used by swarm managers are encrypted on disk by default.
When Docker restarts, both the TLS key used to encrypt communication among swarm nodes, and the key used to encrypt and decrypt Raft logs on disk, are loaded into each manager node’s memoryWe might not want stopped managers to automatically rejoin a swarm as they could decrypt information etc. And we probably dont want to automatically allow restores of swarm backups in case of accidentally overwriting a swarm.
The auto-lock feature allows you to take ownership of these keys and to require manual unlocking of your managers. 

Autolock is not enabled by default - to enable:
  docker swarm init --autolock            <-- at creation time
or:
  docker swarm update --autolock=true     <-- retrospective enable autolock

This generates a password:
  docker swarm update --autolock=true     <-- retrospective enable autolock
    Swarm updated.
  To unlock a swarm manager after it restarts, run the `docker swarm unlock`
  command and provide the following key:

    SWMKEY-1-46k4GTnQ2BhKLEjwKwaEx+QB2Cf7UT8d8yWMXj/27q8

  Please remember to store this key in a password manager, since without it you
  will not be able to restart the manager.

Now if we restart docker on a manager node - we need to specify the password before the manager can rejoin the swarm:
  systemctl restart docker

And now if i try to update the swarm I get an error message telling me I need to unlock it:
  docker swarm update --autolock=true
  Error response from daemon: Swarm is encrypted and needs to be unlocked before it can be used. Please use "docker swarm unlock" to          unlock it.
  
  docker node ls
Error response from daemon: Swarm is encrypted and needs to be unlocked before it can be used. Please use "docker swarm unlock" to unlock it.

We can unlock it by:  
  docker swarm unlock
Please enter unlock key:

After which the manager rejoins the swarm and can ls etc

We can change te crertificate expiry time using --cert-expiry eg:
  docker swarm update --cert-expiry 48h
  Swarm updated.


  


===============================
Containerising an app
===============================
Analysing a Dockerfile:

  FROM alpine                               <-- Use Apline latest image - Creates a layer

  LABEL maintainer="gareth jones"           <-- creates a lable - no layer, just adds metadata to json manifest

  COPY . /src                               <-- copy content from host to container - Creates a layer

  WORKDIR /src                              <-- sets container workng dir - no layer, just adds metadata to json manifest

  RUN apk add --update nodejs nodejs-npm    <-- installs npm - Creates a layer

  EXPOSE 8080                               <-- exposes port 8080 - no layer, just adds metadata to json manifest

  ENTRYPOINT ["node", "./app.js"]           <-- specify command for container to run - no layer, just adds metadata to json manifest

Lets build and run this:
  docker build -t garethjones76/appbuild .
or
  docker build -t psweb .
Now we cam list these:
  docker image ls
    REPOSITORY                                     TAG                 IMAGE ID            CREATED             SIZE
    garethjones76/appbuild                         latest              e5f93bb6b678        4 minutes ago       54.5 MB
    psweb                                          latest              e5f93bb6b678        4 minutes ago       54.5 MB
Now we can run it
  docker container run -d --name web1 -p 8080:8080 psweb
 
We can see it running:
  docker ps
    CONTAINER ID        IMAGE               COMMAND             CREATED              STATUS              PORTS                    NAMES
    5ab7bc3ac6bf        psweb               "node ./app.js"     About a minute ago   Up About a minute   0.0.0.0:8080->8080/tcp   web1
And we can hit the app on http://<host>:8080
  http://192.168.209.101:8080/


=====================
Mutli-stage builds
=====================
Multi-stage builds are a new feature requiring Docker 17.05 used to optimize Dockerfiles while keeping them easy to read and maintain
Keeping the image size down is a best practise (efficientcy ans ecurity). Each instruction in the Dockerfile adds a layer to the image, and one must clean up any uneeded artifacts before moving on to the next layer. Each stage (test, buil, release) has differnet requiements so it was a challenge to keep the layers as small as possible and to ensure that each layer has the artifacts it needs from the previous layer and nothing else. A common pracise was to have one Dockerfile for development (which contained everything needed to build your application), and a slimmed-down one to use for production, which only contained your application and exactly what was needed to run it.

Multi-stage build files helps resolve this issue.

With multi-stage builds, you use multiple FROM statements in your Dockerfile. Each FROM instruction can use a different base, and each of them begins a new stage of the build - each stage is given an integer number starting at 0 and each stage can be named. Artifacts can be selectively copyied from one stage to another, leaving behind everything not required in the final image

eg:
  Dockerfile:

  FROM golang:1.7.3
  WORKDIR /go/src/github.com/alexellis/href-counter/
  RUN go get -d -v golang.org/x/net/html  
  COPY app.go .
  RUN CGO_ENABLED=0 GOOS=linux go build -a -installsuffix cgo -o app .  <-- build the app 

  FROM alpine:latest  
  RUN apk --no-cache add ca-certificates
  WORKDIR /root/
  COPY --from=0 /go/src/github.com/alexellis/href-counter/app .         <-- copy build artefact created in stage 0 to this image
  CMD ["./app"]  

To build this use:

  docker build -t garethjones76/myimage:latest .

or better - use named stages:
  Dockerfile:
  
  FROM golang:1.7.3 as builder                                                <-- name this stage "builder"
  WORKDIR /go/src/github.com/alexellis/href-counter/
  RUN go get -d -v golang.org/x/net/html  
  COPY app.go    .
  RUN CGO_ENABLED=0 GOOS=linux go build -a -installsuffix cgo -o app .        <-- build the app

  FROM alpine:latest  
  RUN apk --no-cache add ca-certificates
  WORKDIR /root/
  COPY --from=builder /go/src/github.com/alexellis/href-counter/app .         <-- copy from "builder" stage to 
  CMD ["./app"] 


NB - build stage uses golang as te base image - release stage uses alpine as the base image which is much smaller and more secure










# Simple web app for Pluralsight courses and Docker Deep Dive book
This is a quick and dirty node.js app cobbled together for the purposes of demonstrating how to Dockerize/containerize a simple app.

**This app is not maintained and WILL be full of vulnerbilities. Use at own risk**

Exposes web server on port 8080 as per ./app.js

See **Dockerfile** for more details
