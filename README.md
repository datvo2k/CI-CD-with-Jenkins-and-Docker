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
        IdentityFile ~/.ssh/@FileYouWasCreated
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



