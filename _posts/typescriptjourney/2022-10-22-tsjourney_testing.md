---
title: "4. Typescript Journey - Testing"
date: 2022-10-22T06:00:00-07:00
draft: false
categories: ["learning","typescript","Typescript Journey"]
description: "How do we test our typescript app?"
layout: post
---
## Goal

Ability to test Typescript app using Jest

[Source Code](https://github.com/two4suited/TypescriptJourney/tree/testing)

## Create Package.json

Up until now we didn't really need a package.json for our project, but to make our commands easier and give our project some metadata, let's create one

```bash
npm init -y
```
This will give us a package.json file that looks something like this.

```json
{
  "name": "TypescriptJourney",
  "version": "1.0.0",
  "description": "Code Repo for Journey to learn Typescript",
  "main": "index.js",
  "scripts": {
    "test": "echo \"Error: no test specified\" && exit 1"
  },
  "repository": {
    "type": "git",
    "url": "git+https://github.com/two4suited/TypescriptJourney.git"
  },
  "keywords": [],
  "author": "",
  "license": "ISC",
  "bugs": {
    "url": "https://github.com/two4suited/TypescriptJourney/issues"
  },
  "homepage": "https://github.com/two4suited/TypescriptJourney#readme"
}

```
We can see in the scripts are there is a key for tests, we will be modify that later to make it run our typescript tests

**Note a package.json in .Net holds information about nuget packages that we have installed in our project**

## Install ts-jest

We are going to install ts-jest so that we can run native typescript tests and not have to convert our typescript into javascript

```bash
npm i -D jest ts-jest @types/jest
```

We will also add this file to the end of our Dockerfile, so we don't have to run this everytime we start up the Environment

We also updated the version of node to 18

Here is the full docker file
```
ARG VARIANT=18-buster
FROM mcr.microsoft.com/vscode/devcontainers/typescript-node:${VARIANT}
RUN apt-get update && export DEBIAN_FRONTEND=noninteractive \
    && apt-get -y install git
RUN npm install -g ts-node typescript '@types/node'
RUN npm install -g jest ts-jest '@types/jest'
```

## Setup ts-jest

Initialize, This will create a jest.config.ts file
```bash
npx ts-jest config:init
```

Modify the package.json so we can run ts-node test and it will know what to do and where to find the test files.

```json
"scripts": {
    "test": "jest"
  },
```


## Create Tests

Let's modify our <b>index.ts</b> file to create two simple functions for adding two numbers and multiplying two numbers

```typescript
export function add(num1: number, num2: number): number {
    return num1+num2 
}
export function multiply(num1: number,num2:number): number {
    return num1 * num2
}
```
We would like to be able to write tests to see if our functions are correct.

Let's create a tests folder and create a file called index.tests.ts
```bash
mkdir tests
cd tests
touch index.test.ts
```

Add the following code to your <i>index.test.ts</i>
```typescript
import {add,multiply} from "../src/index"

describe("test add function", () =>{
    it("should return 2 for 1+1",() =>{
        expect(add(1,1)).toBe(2);
    }),
    it("should return 3 for 1+2",() =>{
        expect(add(1,2)).toBe(3);
    })
})

describe("test multiply function", () =>{
    it("should return 1 when 1*10",() =>{
        expect(multiply(1,10)).toBe(10)
    }),
    it("should be 4 when 2*2", ()=> {
        expect(multiply(2,2)).toBe(4)
    }),
    it("should be 0 when 0*2", ()=> {
        expect(multiply(0,2)).toBe(0)
    })

})
```
## Run Tests
```bash
npm test
```

You should see the following output.
```bash
> TypescriptJourney@1.0.0 test
> jest

 PASS  tests/index.test.ts (12.736 s)
  test add function
    ✓ should return 2 for 1+1 (12 ms)
    ✓ should return 3 for 1+2 (3 ms)
  test multiply function
    ✓ should return 1 when 1*10 (1 ms)
    ✓ should be 4 when 2*2 (2 ms)
    ✓ should be 0 when 0*2 (4 ms)

Test Suites: 1 passed, 1 total
Tests:       5 passed, 5 total
Snapshots:   0 total
Time:        13.286 s
Ran all test suites.
```

## Note
I was getting the following warning when I ran npm test
```
ts-jest[config] (WARN) message TS151001: If you have issues related to imports, you should consider setting `esModuleInterop` to `true` in your TypeScript configuration file (usually `tsconfig.json`). See https://blogs.msdn.microsoft.com/typescript/2018/01/31/announcing-typescript-2-7/#easier-ecmascript-module-interoperability for more information.
```

To solve this I moved the tsconfig.json to the root directory

[Next - 5. Typescript Journey - Visual Studio Code Testing](https://brianpsheridan.com/learning/typescript/typescript%20journey/2022/10/23/tsjourney_vscode.html)