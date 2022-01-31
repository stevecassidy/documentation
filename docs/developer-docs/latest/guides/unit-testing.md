---
title: Unit Testing - Strapi Developer Docs
description: Learn in this guide how you can run basic unit tests for a Strapi application using a testing framework.
sidebarDepth: 2
canonicalUrl: https://docs.strapi.io/developer-docs/latest/guides/unit-testing.html
---

# Unit testing


In this guide we will see how you can run basic unit tests for a Strapi application using a testing framework.

::: tip
In this example we will use [Jest](https://jestjs.io/) Testing Framework with a focus on simplicity and
[Supertest](https://github.com/visionmedia/supertest) Super-agent driven library for testing node.js HTTP servers using a fluent API
:::

:::caution
Please note that this guide will not work if you are on Windows using the SQLite database due to how windows locks the SQLite file.
:::

## Install test tools

`Jest` contains a set of guidelines or rules used for creating and designing test cases - a combination of practices and tools that are designed to help testers test more efficiently.

`Supertest` allows you to test all the `api` routes as they were instances of [http.Server](https://nodejs.org/api/http.md#http_class_http_server)

`sqlite3` is used to create an on-disk database that is created and deleted between tests.

:::: tabs card

::: tab yarn
`yarn add --dev jest supertest sqlite3`
:::

::: tab npm
`npm install jest supertest sqlite3 --save-dev`
:::
::::

Once this is done add this to `package.json` file

add `test` command to `scripts` section

```json
  "scripts": {
    "develop": "strapi develop",
    "start": "strapi start",
    "build": "strapi build",
    "strapi": "strapi",
    "test": "jest --forceExit --detectOpenHandles"
  },
```

and add those line at the bottom of file

```json
  "jest": {
    "testPathIgnorePatterns": [
      "/node_modules/",
      ".tmp",
      ".cache"
    ],
    "testEnvironment": "node"
  }
```

Those will inform `Jest` not to look for test inside the folder where it shouldn't.

## Introduction

### Testing environment

Test framework must have a clean empty environment to perform valid test and also not to interfere with current database.

Once `jest` is running it uses the `test` [environment](/developer-docs/latest/setup-deployment-guides/configurations/optional/environment.md) (switching `NODE_ENV` to `test`)
so we need to create a special environment setting for this purpose.
Create a new config for test env `./config/env/test/database.json` and add the following value `"filename": ".tmp/test.db"` - the reason of that is that we want to have a separate sqlite database for tests, so our test will not touch real data.
This file will be temporary, each time test is finished, we will remove that file that every time tests are run on the clean database.
The whole file will look like this:

**Path —** `./config/env/test/database.json`

```json
{
  "defaultConnection": "default",
  "connections": {
    "default": {
      "connector": "bookshelf",
      "settings": {
        "client": "sqlite",
        "filename": ".tmp/test.db"
      },
      "options": {
        "useNullAsDefault": true,
        "pool": {
          "min": 0,
          "max": 1
        }
      }
    }
  }
}
```


### Test strapi instance

In order to test anything we need to have a strapi instance that runs in the testing eviroment, basically we want to get instance of strapi app as an object, similar like creating an instance for [process manager](process-manager.md).

These tasks require adding some files - let's create a folder `tests` 
where all the tests will be put and inside it, next to folder `helpers` 
where main Strapi helper will be in file `strapi.js`.  This file defines 
two helper functions that will be run before and after all of our
 tests.  They create the Strapi instance and store it in a global 
variable alongside a reference to the internal server that we'll use for 
testing.

**Path -** `./tests/helpers/strapi.js`

```js
const strapi = require('@strapi/strapi')
const fs = require('fs');


const setupStrapi = async () => {
    global.strapiInstance = await strapi().load(); 
    global.strapiInstance.server.mount()    
    global.strapiServer = global.strapiInstance.server.app.callback();
  }
  
  const teardownStrapi = () => {
    global.strapiInstance.destroy().then(() => {
        //delete test database after all tests
        const dbSettings = global.strapiInstance.config.get('database.connections.default.settings');
        if (dbSettings && dbSettings.filename) {
            const tmpDbFile = `${__dirname}/../${dbSettings.filename}`;
            if (fs.existsSync(tmpDbFile)) {
                fs.unlinkSync(tmpDbFile);
            }
        }
    })
  }
  

module.exports = {setupStrapi, teardownStrapi}
```

We need a main entry file for our tests, one that will also test our 
helper file.

**Path —** `./tests/app.test.js`

```js
const {setupStrapi, teardownStrapi} = require('./helpers/strapi')

/** this code is called once before any test is called */
beforeAll(setupStrapi);

/** this code is called once before all the tested are finished */
afterAll(teardownStrapi);

it('strapi instance is defined', () => {
  expect(global.strapiInstance).toBeDefined();
});

```

Actually this is all we need for writing unit tests. Just run `yarn test` and see a result of your first test

```bash
$ yarn test
yarn run v1.22.17
$ jest --forceExit --detectOpenHandles
 PASS  tests/app.test.js (7.391 s)
  ✓ strapi instance is defined (1 ms)

Test Suites: 1 passed, 1 total
Tests:       1 passed, 1 total
Snapshots:   0 total
Time:        7.619 s
Ran all test suites.
✨  Done in 14.71s.
```

:::tip
If you receive a timeout error for Jest, please add the following line right before the `beforeAll` method in the `app.test.js` file: `jest.setTimeout(15000)` and adjust the milliseconds value as you need.
:::

### Testing basic endpoint controller.

::: tip
In the example we'll use and example `Hello world` `/api/hello` endpoint from [controllers](/developer-docs/latest/development/backend-customization/controllers.md) section.
<!-- the link below is reported to have a missing hash by the check-links plugin, but everything is fine 🤷 -->
:::

Some might say that API tests are not unit but limited integration tests, regardless of nomenclature, let's continue with testing first endpoint.

We'll test if our endpoint works properly and route `/api/hello` does return `Hello World`

Let's create a separate test file where `supertest` will be used to check if endpoint works as expected.

**Path —** `./tests/hello/hello.test.js`

```js
const request = require('supertest');
const {setupStrapi, teardownStrapi} = require('../helpers/strapi')

beforeAll(setupStrapi);
afterAll(teardownStrapi);

it('should return hello world', async () => {
  await request(global.strapiServer) // app server is an instance of Class: http.Server
    .get('/api/hello')
    .expect(200) // Expect response http code 200
    .then(data => {
      expect(data.text).toBe('Hello World!'); // expect the response text
    });
});


```
Every test file that you write will have this structure.  Test file names need to include `.test.` to be found by Jest. 

Now run `yarn test` which should find both test files:

```bash
$ yarn run v1.22.17
$ jest --forceExit --detectOpenHandles
 PASS  tests/app.test.js
 PASS  tests/hello/hello.test.js
  ● Console

    console.log
      [2022-01-31 11:39:07.914] http: GET /api/hello (17 ms) 200

      at Console.log (node_modules/winston/lib/winston/transports/console.js:79:23)


Test Suites: 2 passed, 2 total
Tests:       2 passed, 2 total
Snapshots:   0 total
Time:        5.284 s
Ran all test suites.
```

:::tip
If you receive an error `Jest has detected the following 1 open handles potentially keeping Jest from exiting`  **TODO: how to deal with this!**
:::

### Testing `auth` endpoint controller.

In this scenario we'll test authentication login endpoint with two tests

1. Test `/auth/local` that should login user and return `jwt` token
2. Test `/users/me` that should return users data based on `Authorization` header

**Path —** `./tests/user/index.js`

```js
const request = require('supertest');

// user mock data
const mockUserData = {
  username: 'tester',
  email: 'tester@strapi.com',
  provider: 'local',
  password: '1234abc',
  confirmed: true,
  blocked: null,
};

it('should login user and return jwt token', async () => {
  /** Creates a new user and save it to the database */
  await strapi.plugins['users-permissions'].services.user.add({
    ...mockUserData,
  });

  await request(strapi.server) // app server is an instance of Class: http.Server
    .post('/auth/local')
    .set('accept', 'application/json')
    .set('Content-Type', 'application/json')
    .send({
      identifier: mockUserData.email,
      password: mockUserData.password,
    })
    .expect('Content-Type', /json/)
    .expect(200)
    .then(data => {
      expect(data.body.jwt).toBeDefined();
    });
});

it('should return users data for authenticated user', async () => {
  /** Gets the default user role */
  const defaultRole = await strapi.query('role', 'users-permissions').findOne({}, []);

  const role = defaultRole ? defaultRole.id : null;

  /** Creates a new user an push to database */
  const user = await strapi.plugins['users-permissions'].services.user.add({
    ...mockUserData,
    username: 'tester2',
    email: 'tester2@strapi.com',
    role,
  });

  const jwt = strapi.plugins['users-permissions'].services.jwt.issue({
    id: user.id,
  });

  await request(strapi.server) // app server is an instance of Class: http.Server
    .get('/users/me')
    .set('accept', 'application/json')
    .set('Content-Type', 'application/json')
    .set('Authorization', 'Bearer ' + jwt)
    .expect('Content-Type', /json/)
    .expect(200)
    .then(data => {
      expect(data.body).toBeDefined();
      expect(data.body.id).toBe(user.id);
      expect(data.body.username).toBe(user.username);
      expect(data.body.email).toBe(user.email);
    });
});
```

Then include this code to `./tests/app.test.js` at the bottom of that file

```js
require('./user');
```

All the tests above should return an console output like

```bash
➜  my-project git:(master) yarn test

yarn run v1.13.0
$ jest --forceExit --detectOpenHandles
[2020-05-27T08:30:30.811Z] debug GET /hello (10 ms) 200
[2020-05-27T08:30:31.864Z] debug POST /auth/local (891 ms) 200
 PASS  tests/app.test.js (6.811 s)
  ✓ strapi is defined (3 ms)
  ✓ should return hello world (54 ms)
  ✓ should login user and return jwt token (1049 ms)
  ✓ should return users data for authenticated user (163 ms)

Test Suites: 1 passed, 1 total
Tests:       4 passed, 4 total
Snapshots:   0 total
Time:        6.874 s, estimated 9 s
Ran all test suites.
✨  Done in 8.40s.
```
