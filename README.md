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
![results](https://github.com/datvo2k/CI-CD-with-Jenkins-and-Docker/pic/localhost_test_docker.png)

