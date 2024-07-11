
docker sources update
``` 
==https://mirror.ccs.tencentyun.com
vi //etc/containers/registries.conf

#unqualified-search-registries = ["registry.access.redhat.com", "registry.redhat.io", "docker.io"]
unqualified-search-registries = ['docker.io']

[[registry]]
prefix = "docker.io"
# This will set the docker registry mirror of a chinese university.
# DON'T use it unless you have a network connection issue and you trust the mirror provider.
location = "mirror.ccs.tencentyun.com"

```
```
git clone https://github.com/ianmiell/simple-dockerfile
[root@VM-16-14-centos simple-dockerfile]# docker run -it $(docker build -q .)
Emulate Docker CLI using podman. Create /etc/containers/nodocker to quiet msg.
Emulate Docker CLI using podman. Create /etc/containers/nodocker to quiet msg.
Hello world
[root@VM-16-14-centos simple-dockerfile]# echo $(docker build -q .)
Emulate Docker CLI using podman. Create /etc/containers/nodocker to quiet msg.
17cee55c5757ec78edcfc0371c023ea56f193ff4af7542063b8d9608ee6b1e3e
[root@VM-16-14-centos simple-dockerfile]# docker images
Emulate Docker CLI using podman. Create /etc/containers/nodocker to quiet msg.
REPOSITORY                TAG         IMAGE ID      CREATED        SIZE
<none>                    <none>      c60272265270  4 hours ago    49.5 MB
<none>                    <none>      17cee55c5757  4 hours ago    206 MB
quay.io/podman/hello      latest      5dd467fce50b  6 weeks ago    787 kB
docker.io/library/python  3.7-alpine  1bac8ae77e4a  11 months ago  49.5 MB
docker.io/library/ubuntu  14.04       13b66b487594  3 years ago    206 MB
[root@VM-16-14-centos simple-dockerfile]# docker run -it 17cee55c5757
Emulate Docker CLI using podman. Create /etc/containers/nodocker to quiet msg.
Hello world
[root@VM-16-14-centos simple-dockerfile]#


docker rmi -f c60272265270

```  

```
centos stream release9:
yum install docker
curl -L https://github.com/docker/compose/releases/download/v2.7.0/docker-compose-$(uname -s)-$(uname -m) -o /usr/local/bin/docker-compose
chmod 755 /usr/local/bin/docker-compose
docker-compose --version
yum update
--yum install docker-compose-plugin
yum install podman-remote
ystemctl enable --now podman.socket
yum install podman-docker
podman-remote info
ls -al /var/run/docker.sock


[root@VM-16-14-centos single-dev-env]# docker compose -f ./compose-dev.yaml  up
or
docker build .

```

```

FROM centos:centos9
MAINTAINER The CentOS Project

RUN yum -y update; yum clean all
RUN yum -y install epel-release; yum clean all
RUN yum -y install python-pip; yum clean all

ADD . /src

RUN cd /src; pip install -r requirements.txt

EXPOSE 8080

CMD ["python", "/src/index.py"]

```
