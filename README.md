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

해당 repository 를 clone 합니다.

`git clone https://github.com/jgm0710/jenkins-docker.git`

jenkins-docker directory 가 생성되었다면 해당 directory 로 이동합니다.

<img width="698" alt="스크린샷 2022-04-21 오후 3 02 27" src="https://user-images.githubusercontent.com/62986636/164385414-e121ec75-d431-4ea4-9715-0f09426c19be.png">

docker ps 명령어를 사용하여 실행 중인 container 가 있는지 확인할 수 있습니다.

docker-compose up -d 명령어를 사용하여 docker-compose.yml 파일을 실행합니다.

```
docker-compose up -d
```

ubuntu pc 에 docker-compose 실행 파일이 없는 경우 다음과 같은 안내가 나타납니다.

<img width="698" alt="스크린샷 2022-04-21 오후 3 03 21" src="https://user-images.githubusercontent.com/62986636/164385874-6b07fc09-4e9a-4f8b-9326-dfdabc91dba4.png">

권장하는 apt install 명렁어를 사용하여 docker-compose 를 download 해 줍니다.

```
sudo apt install docker-compose
```

<img width="1130" alt="스크린샷 2022-04-21 오후 3 03 31" src="https://user-images.githubusercontent.com/62986636/164385918-9297d567-4ceb-4060-9482-47fe735e19b5.png">

doker-compose 설치가 완료 되었고, docker-compose up 명령어를 사용하여 jenkins container 를 실행했다면, 

docker ps 명령어를 사용하여 컨테이너가 정상적으로 구동되고 있는지 확인합니다.


<img width="745" alt="스크린샷 2022-04-21 오후 3 13 35" src="https://user-images.githubusercontent.com/62986636/164387320-cb89e529-d127-4241-81c6-b05a0e096fd4.png">

jenkins 컨테이너가 구동 중인 ubuntu pc 의 9090 port 에 접속하여, jenkins 에 접속합니다.

해당 실습은 aws ec2 instance 에서 진행되고 있어 ip host 를 통해 접속합니다.
9090 port 를 통해 jenkins 접속이 불가능한 경우 보안 그룹에 tcp 9090 port 에 대해 인바운드 설정이 정상적으로 되어 있는지 확인합니다.

개인 PC 에서 설정을 진행하는 경우 http://localhost:9090  에 접속합니다.

Jenkins 가 정상적으로 구동되고 있다면, 다음의 화면이 나타납니다.

<img width="1021" alt="스크린샷 2022-04-21 오후 3 15 20" src="https://user-images.githubusercontent.com/62986636/164387787-5f284a28-86bb-4832-8081-d7b0eee0e874.png">

비밀번호의 경우 jenkins 가 구동되는 시점에 임의로 설정됩니다.

docker 를 사용하여 구동하지 않은 경우 첫 실행 시 password log 가 나타났겠지만, 우리는 docker-compose up -d 백그라운드 실행 명령어를 사용하여 jenkins container 를 구동했기 때문에 다음의 방법으로 비밀 번호를 확인합니다.

docker ps 명령어를 사용하여 구동 중인 jenkins container 의 이름을 확인합니다.

docker logs [container name] 의 형태로 container log 를 확인할 수 있습니다.

우리의 jenkins container 의 이름은 jenkins 로 설정되어 있으므로, 다음의 명령어를 사용하여 container log 를 확인합니다.

```
docker logs jenkins
```

<img width="750" alt="스크린샷 2022-04-21 오후 3 17 14" src="https://user-images.githubusercontent.com/62986636/164388103-d73e77b4-c2e3-4957-9cd8-7e1a8f7d47fb.png">

나타난 로그를 찾아보면 다음의 문구를 찾을 수 있습니다.

<img width="750" alt="스크린샷 2022-04-21 오후 3 17 28" src="https://user-images.githubusercontent.com/62986636/164388371-e7923d4d-a54f-4f70-8309-ae9c622944f8.png">

위 부분에 나타나 있는 password 를 copy&paste 하여 jenkins 에 입력합니다.

<img width="691" alt="스크린샷 2022-04-21 오후 3 17 51" src="https://user-images.githubusercontent.com/62986636/164388499-36d38f3c-c823-40fd-ac29-06716fbddef1.png">

Install suggested plugins 




