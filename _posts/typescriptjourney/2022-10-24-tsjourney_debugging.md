---
title: "6. Typescript Journey - Debugging"
date: 2022-10-23T21:00:00-07:00
draft: false
categories: ["learning","typescript","Typescript Journey"]
description: "How do we debug our typescript app?"
layout: post
---
## Goal

The ability to use the built in debug tools to debug Typescript programs

[Source Code](https://github.com/two4suited/TypescriptJourney/tree/debug)

## Setup

Install the Typescript Debugger in visual studio code

Add it to devcontainer.json
```json
{
	"build": {"dockerfile": "Dockerfile"},
	"customizations": {
		"vscode": {
		  "extensions": [
			"dbaeumer.vscode-eslint",
			"Orta.vscode-jest",
			"kakumei.ts-debug"
		]
		}
	  },	
	// Use 'forwardPorts' to make a list of ports inside the container available locally.
	 "forwardPorts": [3000]	
}
```
You will then want to select the .ts file you want to debug, click the debug toolbar button and then click run and debug, in this menu you can choose TS Debug

If this does not work you need to modify set it up manually
```bash
mkdir .vscode
cd .vscode
touch launch.json
```
Add following to launch.json
```json
{   
    "version": "0.2.0",
    "configurations": [
        {
            "name": "Debug using ts-node",
            "type": "node",
            "request": "launch",
            "args": [
                "${relativeFile}"
            ],
            "runtimeArgs": [
                "-r",
                "ts-node/register"
            ],
            "cwd": "${workspaceRoot}",
            "protocol": "inspector",
            "internalConsoleOptions": "openOnSessionStart"
        }
    ]
}
```
You can then modify your index.ts file to call one of the functions and write the output to the console

```typescript
export function add(num1: number, num2: number): number {
    return num1+num2 
}
export function multiply(num1: number,num2:number): number {
    return num1 * num2
}

let addedNum = add(1,2)
console.log(addedNum)
```
Add a breakpoint at the **console.log(addednum)** line and then __Run and Debug__ icon and then the grean triangle nex to Debug using ts-node

You will then see all the Variables and it will stop at the breakpoint you set.  The __addedNum__ variable should be 3


