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

Jenkins container 에 대한 volume mouting 설정


```
volumes:
      - ./jenkins_home:/var/jenkins_home
      - /var/run/docker.sock:/var/run/docker.sock
      - /application:/application
```

### install_docker.sh

### .gitignore

## 실행 설명
