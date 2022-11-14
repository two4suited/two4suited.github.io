---
title: "9. Typescript Journey - Express API Docker"
date: 2022-11-13T06:00:00-07:00
draft: false
categories: ["learning","typescript","Typescript Journey"]
description: "Let's run an API in Express using Typescript in Docker"
layout: post
---
[Series Posts](https://brianpsheridan.com/categories.html#typescript-journey)

[Source Code](https://github.com/two4suited/TypescriptJourney/tree/expressapidocker)

## Goal
Run an Express API in Docker

## Setup

We will building off the this [branch](https://github.com/two4suited/TypescriptJourney/tree/expressapi)

You will need download and install [Docker Desktop](https://www.docker.com/products/docker-desktop/).  

## Create Dockerfile
You will want to create two files a **Dockerfile** and **.dockerignore**.  The .dockerignore makes sure that our node_modules and transpiled js files don't get put into the image.

```bash
touch Dockerfile
touch .dockerignore
```

Here is the contents of the .dockerignore
```
node_modules
dist
```

This uses our base image of node:lts-alpine, we could also use node:latest, but the node:latest is over 1gb and this alpine image is around 250mb.
```Dockerfile
FROM node:lts-alpine
```

Sets the working directory to /app, copies the package.json to that folder and then run npm install to install the modules
```Dockerfile
WORKDIR /app
COPY package*.json ./
RUN npm install
```

Copy the rest of the files to the working directory and run npx tsc to transpile them to js files
```Dockerfile
COPY . .
RUN npx tsc
```

This expose port 3000 in the docker file and runs npm start to start up express
```Dockerfile
EXPOSE 3000
CMD ["npm","start"]
```

Here is the full Dockerfile
```Dockerfile
FROM node:lts-alpine
WORKDIR /app
COPY package*.json ./
RUN npm install
COPY . .
RUN npx tsc
EXPOSE 3000
CMD ["npm","start"]
```

## Build and Run Docker Images

Build the Docker Image, in the root image of your folder and tag it with expressapi
```bash
docker build . -t expressapi
```

Run the docker image.  This will name it personapi and map port 3000 to 3000 on the host machine
```
docker run -p 3000:3000 --name personapi expressapi
```

### Test API
---
You can use curl to test your endpoints.  The server will be running on port 3000

Get All
```bash
curl http://localhost:3000/api/people -i
```
Get By Id
```bash
curl http://localhost:3000/api/people/1 -i
```
Create New Record
```bash
curl -X POST -H 'Content-Type: application/json' -d '{"firstName": "Bob","lastName": "Sheridan","age": 1}' http://localhost:3000/api/people -i
```
Update Existing Record
```bash
curl -X PUT -H 'Content-Type: application/json' -d '{"firstName": "B","lastName": "Sheridan","age": 1}' http://localhost:3000/api/people/1 -i
```
Delete Record
```bash
curl -X DELETE http://localhost:3000/api/people/1 -i
```

## Cleanup

Once you have tested your api you can clean it up

Kill Running Docker Image
```
docker kill personapi
```
Remove Container
```
docker container rm personapi
```
Remove Docker Image
```
docker image rm expressapi
```
