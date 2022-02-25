# CI-CD-with-Jenkins-and-Docker

## Build docker image on local
Write Dockerfile. Add source code and config ngnix.
```
#get the latest alpine image from node registry
FROM node:alpine AS build-stage

#set the working directory
WORKDIR /app

#copy the package and package lock files
#from local to container work directory /app
COPY package*.json /app/

#Run command npm install to install packages
RUN npm install

#copy all the folder contents from local to container
COPY . .

#create a react production build
RUN npm run build

#get the latest alpine image from nginx registry
FROM nginx:alpine

#we copy the output from first stage that is our react build
#into nginx html directory where it will serve our index file
COPY --from=build-stage /app/build/ /usr/share/nginx/html
```
Then we build Dockerfile we have been written.
```
docker build -t mine-blog:dev .
```
When you have many images, it becomes difficult to know which image is what. So you have to tag your image.
Let running docker on localhost, we allow docker to publish port.
```
docker container run -p 4001:80 9755de4bfb81
```
this command i running port 4001.
As a result
![results](https://github.com/datvo2k/CI-CD-with-Jenkins-and-Docker/blob/main/pic/localhost_test_docker.png)

## Create SSH key
```
ssh-keygen -t rsa -b 4096 -C "your_email@gmail.com"
```
And the output:
```
Generating public/private rsa key pair.
Enter file in which to save the key (/home/datvo2k/.ssh/id_rsa): gucciKeyForEveryThing2
Enter passphrase (empty for no passphrase):
Enter same passphrase again:
Your identification has been saved in gucciKeyForEveryThing2
Your public key has been saved in gucciKeyForEveryThing2.pub
The key fingerprint is:
SHA256:NGwc04M/SJf8L4voN7K/RWmatx725t8mN5/jeD5QShE datvo2k@Brian_Desktop
The key's randomart image is:
+---[RSA 4096]----+
|        o+ . E.  |
|       ooo*  .   |
|       .*+ o  .  |
|       o..o .o . |
|        S  .=.o  |
|           =.o.  |
|         .o.=o.  |
|        o ++.++==|
|       .o=o+o+*OO|
+----[SHA256]-----+
```
Check file exist or not:
```
ls -l ~/.ssh
```
Next step you must copy the public key to your server. Final step you should manage SSH keys.
```
touch ~/.ssh/config
```
And edit this file was created.
```
Host @nameHostYouRemote
        HostName @Ip
        IdentityFile ~/.ssh/id_rsa
        User @userRemote
```
## Install on VPS
In this repo, i using digitalocean VPS(10$ / 1month) and run as root.
Run this command line
```
apt-get update -y && apt-get upgrade 
```
After update, we install fail2ban 
```
apt install fail2ban
```
When the installation is completed, we must verify it by checking the status of the service:
```
systemctl status fail2ban
```
Create a copy of the configuration file name jail.local and edit file in a text editor
```
cp /etc/fail2ban/jail.conf /etc/fail2ban/jail.local
nano /etc/fail2ban/jail.local
```
Update the intial setting:
```
bantime = 60m
findtime = 5m
maxretry = 7
```
Protect SSH
```
[sshd]
enabled = true
maxretry = 7
bantime = 60m
findtime = 5m
ignoreip  = 127.0.0.1/8 23.30.56.56
```
You need to restart the fail2ban service for changes to take effect:
```
systemctl restart fail2ban
```
Here a few commands:
```
fail2ban-client status sshd             # check status
zgrep 'Ban' /var/log/fail2ban.log*      # look at the full list IPs have been blocked
```
So next step you create folder and run docker-compose.
```
ssh @nameHostYouRemote
```
Into server, organize folder like this:
```
├── docker-compose.yml
├── jenkins
│   ├── Dockerfile
│   └── docker-entrypoint.sh
├── nginx
│   ├── default.conf
│   └── prefix_jenkins.conf
└── ssl
```
In ssl folder, we run command to authenticate ssl:
```
openssl req -x509 -outform pem -out chain.pem -keyout privkey.pem \
  -newkey rsa:4096 -nodes -sha256 -days 3650 \
  -subj '/CN=localhost' -extensions EXT -config <( \
   printf "[dn]\nCN=localhost\n[req]\ndistinguished_name = dn\n[EXT]\nsubjectAltName=DNS:localhost\nkeyUsage=digitalSignature\nextendedKeyUsage=serverAuth")
cat chain.pem > fullchain.pem
```
Create file .env and populate according to your environment
```
# docker-compose environment file

# Jenkins Settings
export JENKINS_LOCAL_HOME=./jenkins_home
export JENKINS_UID=1000
export JENKINS_GID=1000
export HOST_DOCKER_SOCK=/var/run/docker.sock

# Nginx Settings
export NGINX_CONF=./nginx/default.conf
export NGINX_SSL_CERTS=./ssl
export NGINX_LOGS=./logs/nginx
```
This all script below is referenced at https://github.com/mjstealey \
Now edit file `docker-compose.yml`
```
version: "3.9"
services:

  jenkins:
    # default ports 8080, 50000 - expose mapping as needed to host
    build:
      context: ./jenkins
      dockerfile: Dockerfile
    container_name: cicd-jenkins
    restart: unless-stopped
    networks:
      - jenkins
    ports:
      - "50000:50000"
    environment:
      # Uncomment JENKINS_OPTS to add prefix: e.g. https://127.0.0.1:8443/jenkins
      #- JENKINS_OPTS="--prefix=/jenkins"
      - JENKINS_UID=${JENKINS_UID}
      - JENKINS_GID=${JENKINS_GID}
    volumes:
      - ${JENKINS_LOCAL_HOME}:/var/jenkins_home
      - ${HOST_DOCKER_SOCK}:/var/run/docker.sock

  nginx:
    # default ports 80, 443 - expose mapping as needed to host
    image: nginx:1
    container_name: cicd-nginx
    restart: unless-stopped
    networks:
      - jenkins
    ports:
      - "8080:80"    # http
      - "8443:443"   # https
    volumes:
      - ${JENKINS_LOCAL_HOME}:/var/jenkins_home
      - ${NGINX_CONF}:/etc/nginx/conf.d/default.conf
      - ${NGINX_SSL_CERTS}:/etc/ssl
      - ${NGINX_LOGS}:/var/log/nginx

networks:
  jenkins:
    name: cicd-jenkins
    driver: bridge
```
Into jenkins folder, edit `Dockerfile`
```
FROM jenkins/jenkins:lts-jdk11

USER root
RUN apt-get update && apt-get install -y lsb-release sudo
RUN curl -fsSLo /usr/share/keyrings/docker-archive-keyring.asc \
  https://download.docker.com/linux/debian/gpg
RUN echo "deb [arch=$(dpkg --print-architecture) \
  signed-by=/usr/share/keyrings/docker-archive-keyring.asc] \
  https://download.docker.com/linux/debian \
  $(lsb_release -cs) stable" > /etc/apt/sources.list.d/docker.list
RUN apt-get update && apt-get install -y docker-ce-cli \
    && apt-get clean \
    && rm -rf /var/lib/apt/lists/*

USER jenkins

# set default user attributes
ENV JENKINS_UID=1000
ENV JENKINS_GID=1000

# add entrypoint script
COPY docker-entrypoint.sh /docker-entrypoint.sh

# normally user would be set to jenkins, but this is handled by the docker-entrypoint script on startup
#USER jenkins
USER root

# bypass normal entrypoint and use custom one
#ENTRYPOINT ["/sbin/tini", "--", "/usr/local/bin/jenkins.sh"]
ENTRYPOINT ["/sbin/tini", "--", "/docker-entrypoint.sh"]
```
and `docker-entrypoint.sh`
```
#!/usr/bin/env bash
set -e

# jenkins user requires valid UID:GID permissions at the host level to persist data
#
# set Jenkins UID and GID values (as root)
usermod -u ${JENKINS_UID} jenkins
groupmod -g ${JENKINS_GID} jenkins
# update ownership of directories (as root)
{
  chown -R jenkins:jenkins /var/jenkins_home
  chown -R jenkins:jenkins /usr/share/jenkins/ref
} ||
{
  echo "[ERROR] Failed 'chown -R jenkins:jenkins ...' command"
}

# allow jenkins to run sudo docker (as root)
echo "jenkins ALL=(root) NOPASSWD: /usr/bin/docker" > /etc/sudoers.d/jenkins
chmod 0440 /etc/sudoers.d/jenkins

# run Jenkins (as jenkins)
sed -i "s# exec java# exec $(which java)#g" /usr/local/bin/jenkins.sh
su jenkins -c 'cd $HOME; export PATH=$PATH:$(which java); /usr/local/bin/jenkins.sh'
```
So must be remember run this for `docker-entryponit.sh`
```
chmod + x docker-entryponit.sh
```
Next `cd ngnix` and write `default.conf`
```
upstream jenkins {
    keepalive 32;              # keepalive connections
    server cicd-jenkins:8080;  # jenkins container ip and port
}

# Required for Jenkins websocket agents
map $http_upgrade $connection_upgrade {
    default upgrade;
    '' close;
}

server {
    listen 80;                    # Listen on port 80 for IPv4 requests
    server_name $host;
    return 301 https://$host:8443$request_uri; # replace '8443' with your https port
}

server {
    listen          443 ssl;      # Listen on port 443 for IPv4 requests
    server_name     $host:8443;   # replace '$host:8443' with your server domain name and port

    # SSL certificate - replace as required with your own trusted certificate
    ssl_certificate /etc/ssl/fullchain.pem;
    ssl_certificate_key /etc/ssl/privkey.pem;

    # logging
    access_log      /var/log/nginx/jenkins.access.log;
    error_log       /var/log/nginx/jenkins.error.log;

    # this is the jenkins web root directory
    # (mentioned in the /etc/default/jenkins file)
    root            /var/jenkins_home/war/;

    # pass through headers from Jenkins that Nginx considers invalid
    ignore_invalid_headers off;

    location ~ "^/static/[0-9a-fA-F]{8}\/(.*)$" {
        # rewrite all static files into requests to the root
        # E.g /static/12345678/css/something.css will become /css/something.css
        rewrite "^/static/[0-9a-fA-F]{8}\/(.*)" /$1 last;
    }

    location /userContent {
        # have nginx handle all the static requests to userContent folder
        # note : This is the $JENKINS_HOME dir
        root /var/jenkins_home/;
        if (!-f $request_filename) {
            # this file does not exist, might be a directory or a /**view** url
            rewrite (.*) /$1 last;
            break;
        }
        sendfile on;
    }

    location / {
        sendfile off;
        proxy_pass         http://jenkins;
        proxy_redirect     default;
        proxy_http_version 1.1;

        # Required for Jenkins websocket agents
        proxy_set_header   Connection        $connection_upgrade;
        proxy_set_header   Upgrade           $http_upgrade;

        proxy_set_header   Host              $host;
        proxy_set_header   X-Real-IP         $remote_addr;
        proxy_set_header   X-Forwarded-For   $proxy_add_x_forwarded_for;
        proxy_set_header   X-Forwarded-Proto $scheme;
        proxy_set_header   X-Forwarded-Host  $host;
        proxy_set_header   X-Forwarded-Port  8443; # replace '8443' with your https port
        proxy_max_temp_file_size 0;

        #this is the maximum upload size
        client_max_body_size       10m;
        client_body_buffer_size    128k;

        proxy_connect_timeout      90;
        proxy_send_timeout         90;
        proxy_read_timeout         90;
        proxy_buffering            off;
        proxy_request_buffering    off; # Required for HTTP CLI commands
        proxy_set_header Connection ""; # Clear for keepalive
    }

}
``` 
And `prefix_jenkins.conf`
```
upstream jenkins {
    keepalive 32;              # keepalive connections
    server cicd-jenkins:8080;  # jenkins container ip and port
}

# Required for Jenkins websocket agents
map $http_upgrade $connection_upgrade {
    default upgrade;
    '' close;
}

server {
    listen 80;                    # Listen on port 80 for IPv4 requests
    server_name $host;
    return 301 https://$host:8443$request_uri; # replace '8443' with your https port
}

server {
    listen          443 ssl;      # Listen on port 443 for IPv4 requests
    server_name     $host:8443;   # replace '$host:8443' with your server domain name and port

    # SSL certificate - replace as required with your own trusted certificate
    ssl_certificate /etc/ssl/fullchain.pem;
    ssl_certificate_key /etc/ssl/privkey.pem;

    # logging
    access_log      /var/log/nginx/jenkins.access.log;
    error_log       /var/log/nginx/jenkins.error.log;

    # this is the jenkins web root directory
    # (mentioned in the /etc/default/jenkins file)
    root            /var/jenkins_home/war/;

    # pass through headers from Jenkins that Nginx considers invalid
    ignore_invalid_headers off;

    location ~ "^/static/[0-9a-fA-F]{8}\/(.*)$" {
        # rewrite all static files into requests to the root
        # E.g /static/12345678/css/something.css will become /css/something.css
        rewrite "^/static/[0-9a-fA-F]{8}\/(.*)" /$1 last;
    }

    location /jenkins/userContent {
        # have nginx handle all the static requests to userContent folder
        # note : This is the $JENKINS_HOME dir
        root /var/jenkins_home/;
        if (!-f $request_filename) {
            # this file does not exist, might be a directory or a /**view** url
            rewrite (.*) /$1 last;
            break;
        }
        sendfile on;
    }

    location /jenkins {
        sendfile off;
        proxy_pass         http://jenkins;
        proxy_redirect     default;
        proxy_http_version 1.1;

        # Required for Jenkins websocket agents
        proxy_set_header   Connection        $connection_upgrade;
        proxy_set_header   Upgrade           $http_upgrade;

        proxy_set_header   Host              $host;
        proxy_set_header   X-Real-IP         $remote_addr;
        proxy_set_header   X-Forwarded-For   $proxy_add_x_forwarded_for;
        proxy_set_header   X-Forwarded-Proto $scheme;
        proxy_set_header   X-Forwarded-Host  $host;
        proxy_set_header   X-Forwarded-Port  8443; # replace '8443' with your https port
        proxy_max_temp_file_size 0;

        #this is the maximum upload size
        client_max_body_size       10m;
        client_body_buffer_size    128k;

        proxy_connect_timeout      90;
        proxy_send_timeout         90;
        proxy_read_timeout         90;
        proxy_buffering            off;
        proxy_request_buffering    off; # Required for HTTP CLI commands
        proxy_set_header Connection ""; # Clear for keepalive
    }

    location / {
        rewrite ^/ $scheme://$host:8443/jenkins permanent; # replace '8443' with your https port
    }

}
```
Next run `source .env` and run `docker-compose up`. Copy the password from the file intialAdminPassword. Then choose `install suggested plugins`. Wait until the plugins are completed installed and create an Admin user for Jenkins.
![jenkins_admin1](https://github.com/datvo2k/CI-CD-with-Jenkins-and-Docker/blob/main/pic/jenkins_1.png)
![jenkins_admin2](https://github.com/datvo2k/CI-CD-with-Jenkins-and-Docker/blob/main/pic/jenkins_3.png)
Finnally here is the default jenkins page.
![jenkins_admin3](https://github.com/datvo2k/CI-CD-with-Jenkins-and-Docker/blob/main/pic/jenkins_2.png)

Next move, add github Oauth. 
![jenkins_admin4](https://github.com/datvo2k/CI-CD-with-Jenkins-and-Docker/blob/main/pic/jenkins_4.png)
And remember add `securityRealm/finishLogin` behind your URL in `Authorization callback URL`. Go to `Dashboard -> Configure Global Security` and add `Client ID`
![jenkins_admin5](https://github.com/datvo2k/CI-CD-with-Jenkins-and-Docker/blob/main/pic/jenkins_6.png)

Next step, we go to jenkins and config pipeline. <br/>
Go to `Dashboard -> Plugin manager` and install some plugins we need.
![jenkins_admin6](https://github.com/datvo2k/CI-CD-with-Jenkins-and-Docker/blob/main/pic/jenkins_7.png)
`Dashboard -> Credentials -> System -> Global credentials` and add credentials you would add. Remember in `password` must be add token for you software.
![jenkins_admin11](https://github.com/datvo2k/CI-CD-with-Jenkins-and-Docker/blob/main/pic/jenkins_12.png)
![jenkins_admin7](https://github.com/datvo2k/CI-CD-with-Jenkins-and-Docker/blob/main/pic/jenkins_13.png)
Now we will create our pipeline. Type the name you want for this Pipeline project
![jenkins_admin8](https://github.com/datvo2k/CI-CD-with-Jenkins-and-Docker/blob/main/pic/jenkins_11.png)
You have to add github credentials and give your github repository url. And save and apply it. Here you have to write your dockerfile for your project. 
![jenkins_admin9](https://github.com/datvo2k/CI-CD-with-Jenkins-and-Docker/blob/main/pic/jenkins_14.png)
![jenkins_admin10](https://github.com/datvo2k/CI-CD-with-Jenkins-and-Docker/blob/main/pic/Screenshot%202022-02-26%20012122.png)
We'll need to tell Jenkins what our stages are, and what to do in each one of them. For this we'll write a Jenkins Pipeline in a `Jenkinsfile`
```
pipeline {
  environment {
    imagename = "brianvo/blog-demo"
    registryCredential = 'docker-hub-webdemo'
    dockerImage = ''
  }
  agent any
  stages {
    stage('Cloning Git') {
      steps {
        checkout scm
      }
    }
    stage('Building image') {
      steps{
        script {
          dockerImage = docker.build imagename
        }
      }
    }
    stage('Deploy Image') {
      steps{
        script {
          docker.withRegistry( '', registryCredential ) {
            dockerImage.push("$BUILD_NUMBER")
             dockerImage.push('latest')
          }
        }
      }
    }
    stage('Remove Unused docker image') {
      steps{
        sh "docker rmi $imagename:$BUILD_NUMBER"
         sh "docker rmi $imagename:latest"
 
      }
    }
  }
}
```
Go to dockerhub and create your repo. Replace `imagename` and `registryCredential ` by your docker repo and ID credentials.
Build project and waiting process done!!!!
![jenkins_admin11](https://github.com/datvo2k/CI-CD-with-Jenkins-and-Docker/blob/main/pic/jenkins_final.png)






