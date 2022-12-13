### Dockerfile

```bash
ARG HOST_UID=1001
ARG HOST_GID=991
FROM jenkins/jenkins:2.349-jdk11
USER root
RUN apt-get -y update && \
 apt-get -y install lsb-release apt-transport-https gnupg2 wget ca-certificates lsb-release software-properties-common make && \
 mkdir -p /etc/apt/keyrings && \
 curl -fsSL https://download.docker.com/linux/ubuntu/gpg | gpg --dearmor -o /etc/apt/keyrings/docker.gpg && \
 echo \
 "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu \
focal stable" | tee /etc/apt/sources.list.d/docker.list > /dev/null && \
 apt-get update && \
 apt-get -y install docker-ce-cli docker-compose-plugin gettext-base

RUN jenkins-plugin-cli --plugins bitbucket sshd junit

ADD .ssh/ .
RUN chown -R jenkins:jenkins /var/jenkins_home

WORKDIR /var/jenkins_home
RUN groupadd docker
RUN usermod -u 1001 jenkins && \
    groupmod -g 991 docker && \
    usermod -aG docker jenkins
USER jenkins

```

### docker-compose.yml
```bash
version: '3.1'
services:
    jenkins:
        build:
            context: ./
            args:
                HOST_UID: 994
                HOST_GID: 991
        restart: unless-stopped
        volumes:
            - /var/run/docker.sock:/var/run/docker.sock
            - /data/jenkins_home:/var/jenkins_home
        ports:
            - 8098:8080
            - 50000:50000
        environment:
            - JENKINS_OPTS="--prefix=/jenkins"

```

### Nginx config
```bash
location ~ /jenkins {
       sendfile off;
       proxy_redirect     off;
       proxy_http_version 1.1;

       proxy_set_header   Connection        $connection_upgrade;
       proxy_set_header   Upgrade           $http_upgrade;
       proxy_set_header   Host              $host;
       proxy_set_header   X-Real-IP         $remote_addr;
       proxy_set_header   X-Forwarded-For   $proxy_add_x_forwarded_for;
       proxy_set_header   X-Forwarded-Proto $scheme;
       proxy_max_temp_file_size 0;

#       #this is the maximum upload size
       client_max_body_size       10m;
       client_body_buffer_size    128k;
#
       proxy_connect_timeout      90;
       proxy_send_timeout         90;
       proxy_read_timeout         90;
       proxy_buffering            off;
       proxy_request_buffering    off; # Required for HTTP CLI commands
       proxy_set_header Connection ""; # Clear for keepalive
       proxy_pass         http://10.10.24.116:8098;

    }
```