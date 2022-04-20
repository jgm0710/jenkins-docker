# Jenkins Docker Setting Guide

- docker 를 사용하여 jenkins 설정 작업을 기록하기 위한 repository 

## 파일 구조 설명

### Dockerfile

- jenkins image 를 dockerhub 에서 다운로드 받고, 내부적으로 docker 를 install 하여 jenkins image 를 build 하기 위한 Dockerfile 

#### Code

```
FROM jenkins/jenkins:lts

USER root

COPY install_docker.sh /install_docker.sh
RUN chmod +x /install_docker.sh
RUN /install_docker.sh

RUN usermod -aG docker jenkins
RUN setfacl -Rm d:g:docker:rwx,g:docker:rwx /var/run/

USER jenkins
```

#### Description

설치 후 도커그룹의 jenkins 계정 생성 후 해당 계정으로 변경

```
RUN usermod -aG docker jenkins
```

/var/run/ directory 에 대해 default group docker 에게 rwx permission 을 부여함을 의미
[ACL 정리 블로그](https://tar-cvzf-studybackup-tar-gz.tistory.com/79)

```
RUN setfacl -Rm d:g:docker:rwx,g:docker:rwx /var/run/
```

User 를 jenkins 로 설정
```
USER jenkins
```


### docker-compose.yml

#### Code

```
version: '3'

services:
  jenkins:
    build:
      context: .
    container_name: jenkins
    user: root
    ports:
      - "9090:8080"
      - "50000:50000"
    volumes:
      - ./jenkins_home:/var/jenkins_home
      - /var/run/docker.sock:/var/run/docker.sock
      - /application:/application
    restart: always
```

### Description

jenkins container 접속 시 root 권한으로 접근할 수 있도록 설정합니다.

`user: root`

jenkins container 는 기본적으로 8080 port 에서 실행되나, spring 프로젝트 등도 8080 port 에서 사용될 수 있으므로 jenkins running port 를 변경합니다.

application 실행 port 를 변경하고, jenkins 실행 포트를 8080 으로 유지하여도 되지만, 명시적으로 볼륨 마운팅하는 것이 변경 시 용이할 것 입니다.

```
    ports:
      - "9090:8080"
      - "50000:50000"
```

Jenkins container 에 대한 volume mount 설정

docker-compose.yml 이 있는 directory 의 path 가 `./` 로 잡힙니다.

```
volumes:
      - ./jenkins_home:/var/jenkins_home
      - /var/run/docker.sock:/var/run/docker.sock
      - /application:/application
```

docker-compose.yml 이 위치하는 directory 의 `jenkins_home` jenkins contaier 의 `/var/jenkins_home` directory 가 마운팅 되어 위치하게 됩니다.

`- ./jenkins_home:/var/jenkins_home`

아래 마운팅 설정은 ubuntu local pc 의 `/var/run/docker.sock` directory 와 jenkins 의 `/var/run/docker.sock` directory 를 마운팅하는 설정입니다.

해당 방식으로 `/var/run/docker.sock` directory 를 일치시켜 줌으로써 기존에 설치된(ubuntu 에 pc 에 설치된) docker 를 사용하여 jenkins 내부적으로 docker 를 사용할 수 있게 됩니다.

해당 방식을 [DooD(Docker out of Docker)](https://aidanbae.github.io/code/docker/dinddood/) 라 합니다. 

`- /var/run/docker.sock:/var/run/docker.sock`

내부적으로 spring 등의 실제 application project 가 위치하도록 만들 volume 설정입니다. 

추후 spring container image 를 build 하여 jenkins 의 `/application` directory 에 위치하도록 합니다.

ubuntu local pc 의 `/applicadtion` directory 와 jenkins container 의 `/application` directory 를 볼륨 마운팅 시킵니다.

`- /application:/application`

### install_docker.sh

jenkins image build 시 jenkins 내부적으로 사용할 프로그램들을 함께 install 하기 위한 실행 파일입니다.

docker, docker-compose 등을 설치합니다.

```
#!/bin/sh

apt-get update

apt-get -y install apt-transport-https \
     apt-utils \
     ca-certificates \
     curl \
     gnupg2 \
     zip \
     unzip \
     acl \
     software-properties-common

curl -fsSL https://download.docker.com/linux/$(. /etc/os-release; echo "$ID")/gpg > /tmp/dkey; apt-key add /tmp/dkey

add-apt-repository \
   "deb [arch=amd64] https://download.docker.com/linux/$(. /etc/os-release; echo "$ID") \
   $(lsb_release -cs) \
   stable" && \

apt-get update

apt-get -y install docker-ce
apt-get -y install docker-compose
```

### .gitignore

배포 환경에서 생기는 `.swp`, `.swn` 등 git repository 에 upload 되지 않길 바라는 대상에 대해서 무시되도록 설정합니다.

## 실행 설명
