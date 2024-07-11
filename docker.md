
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
