---
title: "3. Typescript Journey - Hello World"
date: 2022-10-03T06:00:00-07:00
draft: false
categories: ["learning","typescript","Typescript Journey"]
description: "Walkthrough creating our first app in typescript"
layout: post
---


[**Here is the repo**](https://github.com/two4suited/TypescriptJourney/tree/helloworld)

## Run your app with Node
1. Create src folder
````bash
    mkdir src
    cd src
````
2. Create typescript file
````bash
    touch index.ts
````
3. Initialize tsc
```bash
    tsc --init
```
4. Output "Hello World" to console.  Paste code into index.ts
````typescript
    console.log("Hello World")
````
5. Create .js file from typescript
````bash
    tsc 
````
6. Run your app
````bash
    node index.js
````
7. Remove .js file
````bash
    rm index.js
````

## Run your app with ts-node

Another alternative to run your app is to use ts-node, this does not require creating a js file

1. Install ts-node
````bash
    npm install -g ts-node typescript '@types/node'
````
2. Add ts-node to docker image so you don't have to install everytime.  In your dockerfile add the following line
````
    RUN npm install -g ts-node typescript '@types/node'
````
3. Run your ts file
````bash
    ts-node index.ts
````

## Next
---

[4. Typescript Journey - Testing](https://brianpsheridan.com/learning/typescript/2022/10/22/tsjourney_testing.html)