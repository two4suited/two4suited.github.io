---
title: "8. Typescript Journey - Simple API"
date: 2022-11-06T06:00:00-07:00
draft: false
categories: ["learning","typescript","Typescript Journey"]
description: "Let's run an API in Express using Typescript"
layout: post
---
[Series Posts](https://brianpsheridan.com/categories.html#typescript-journey)

[Source Code](https://github.com/two4suited/TypescriptJourney/tree/expressapi)

## Goal
Use Express to host a API with typescript and also split our code into different files

## Setup

Take the work we did on our express project from [Express Code](https://github.com/two4suited/TypescriptJourney/tree/express)

We need to update the **tsconfig.json** 

Change target to ES2017
```json
 "target": "es2017"
```
Change the rootDir to the src folder
```json
"rootDir":"./src"
```

## Create API

We create the full suite of routes 
- Get All
- Get by ID
- Create New Record
- Update Record
- Delete Record

We are also going to split our files by concern
- Interface - Models 
- Service - Interact with Models and contain Mock Data, does most of the work
- Router - All the endpoints that will call the service file

### Create Models
---
We will be creating a models for a Person, then also a list of people.  

```bash
mkdir src
touch person.interface.ts
```

Create two interfaces, one will used when we create a new person and the other one will extend that and contain the ID

```typescript
export interface BasePerson {
    firstName: string;
    lastName: string;
    age: number
}

export interface Person extends BasePerson {
    id: number;
}
```
### Create People interface
---
Create the interface that will hold a list of having the id as the key value

```bash
touch people.interface.ts
```
```typescript
import { Person } from "./person.interface";

export interface People {
    [key:number]: Person
}
```
### Create Service
---
Create Service file
```bash
touch people.service.ts
```

This service files contains the following
- Imports the two interfaces
- Sets up mock data
- Contains CRUD methods that update that mock data

```typescript
import { BasePerson,Person } from "./person.interface";
import { People } from "./people.interface";

let people: People ={
    1: {
        id: 1,
        firstName: "Brian",
        lastName: "Sheridan",
        age: 46
    },
    2: {
        id: 2,
        firstName: "Heidi",
        lastName: "Sheridan",
        age: 44
    },
}

export const findAll = async () : Promise<Person[]> => Object.values(people);
export const find = async (id: number): Promise<Person> => people[id];
export const create = async (newItem: BasePerson): Promise<Person> => {
    const id = new Date().valueOf();
  
    people[id] = {
      id,
      ...newItem
    };
  
    return people[id];
  };
  export const update = async (
    id: number,
    itemUpdate: BasePerson
  ): Promise<Person | null> => {
    const item = await find(id);
  
    if (!item) {
      return null;
    }
  
    people[id] = { id, ...itemUpdate };
  
    return people[id];
  };
  export const remove = async (id: number): Promise<null | void> => {
    const item = await find(id);
  
    if (!item) {
      return null;
    }
  
    delete people[id];
  };
```
### Create Routes
---
We will be using the express Router to setup our routes that will be the API endpoints that are exposed.  These routes will interact with our service file and read or write data.

Create Router file
```bash
touch people.router.ts
```

These routes contain your standard Http methods and add some error handling for errors that may occur.

```typescript
import express, {Request,Response} from "express"
import * as PeopleService from "./people.service"
import { BasePerson,Person } from "./person.interface"

export const peopleRouter = express.Router();

peopleRouter.get("/",async(req: Request,res:Response) => {
    try{
        const items: Person[] = await PeopleService.findAll();
        res.status(200).send(items)
    } catch(e) {
        res.status(500).send(e instanceof Error ? e.message : "Unknown error.")
    }
});
peopleRouter.get("/:id", async (req: Request, res: Response) => {
    const id: number = parseInt(req.params.id, 10);
  
    try {
      const item: Person = await PeopleService.find(id);
  
      if (item) {
        return res.status(200).send(item);
      }
  
      res.status(404).send("item not found");
    } catch (e) {
        res.status(500).send(e instanceof Error ? e.message : "Unknown error.")
    }
  });
  peopleRouter.post("/", async (req: Request, res: Response) => {
    try {
      const item: BasePerson = req.body;
  
      const newItem = await PeopleService.create(item);
  
      res.status(201).json(newItem);
    } catch (e) {
        res.status(500).send(e instanceof Error ? e.message : "Unknown error.")
    }
  });
  
  // PUT items/:id
  
  peopleRouter.put("/:id", async (req: Request, res: Response) => {
    const id: number = parseInt(req.params.id, 10);
  
    try {
      const itemUpdate: Person = req.body;
  
      const existingItem: Person = await PeopleService.find(id);
  
      if (existingItem) {
        const updatedItem = await PeopleService.update(id, itemUpdate);
        return res.status(200).json(updatedItem);
      }
  
      const newItem = await PeopleService.create(itemUpdate);
  
      res.status(201).json(newItem);
    } catch (e) {
        res.status(500).send(e instanceof Error ? e.message : "Unknown error.")
    }
  });
  
  // DELETE items/:id
  
  peopleRouter.delete("/:id", async (req: Request, res: Response) => {
    try {
      const id: number = parseInt(req.params.id, 10);
      await PeopleService.remove(id);
  
      res.sendStatus(204);
    } catch (e) {
        res.status(500).send(e instanceof Error ? e.message : "Unknown error.")
    }
  });
  ```
### Update index.ts
---
We need to tell express that we are going to be handling json data and also the route for our peopleRouter.

Move index.ts file from root to src folder

Your final index.ts should look like this.
```typescript
import { peopleRouter } from "./people.router";
import express from "express";

const PORT: number = 3000
const app = express();

app.use(express.json())
app.use("/api/people",peopleRouter);

app.listen(PORT, () => {
    console.log(`Listening on port ${PORT}`);
  });
```

### Testing
---
You can use curl to test your endpoints.  The server will be running on port 3000

Run the app
```bash
npm run dev
```

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
---
[Next we will put this app in Docker]()




