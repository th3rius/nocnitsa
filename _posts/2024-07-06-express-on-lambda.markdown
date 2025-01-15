---
layout: post
title: "Hosting Express applications on Lambda"
date: 2024-07-06 21:28:10 -0300
categories: javascript
tags: aws node express serverless lambda
excerpt: >-
  Hosting your Node.js API efficiently while keeping costs low can be a tricky
  task. Here's how I do it.
---

This might sound like an exquisite combination, but my few years using Lambda
to host Express APIs have shown me this is a delightful, headache-free approach
with very little costs, cold boot times and drawbacks.

I love AWS because its flexibility allows for achieving hard to beat prices,
but harvesting such benefits within its [hundreds of
services](https://aws.amazon.com/products/) and a sea of documentation that way
too often is more technical than helpful can be quite the daunting task.
Hopefully this trick up my sleeve will make your journey in the cloud a little
more pleasant.

TL;DR: [here's full code example on
GitHub](https://github.com/th3rius/serverless-express-starter).

## Lambda and HTTP

Lambda by itself is not aware and cannot receive HTTP requests—instead, [API
Gateway](https://docs.aws.amazon.com/apigateway/latest/developerguide/welcome.html)
will be the responsible for triggering our Lambda function whenever a request
comes in.

Conversely, Express cannot interpret these Lambda events. That's where
[`@codegenie/serverless-express`](https://www.npmjs.com/package/@codegenie/serverless-express)
comes into play, allowing us to maintain portability and to make use of the
well-established ecosystem while still making our application serverless.

Typically, I like to split my application entrypoint in three files:

- `app.js`: A function that creates an Express instance;
- `main.js`: Takes an Express instance and spawns a development server;
- `lambda.js`: The Lambda handler that uses our app.

Let's take a closer look:

Firstly, we initialize our express Instance. Separating the Express instance in
its own module gives us the ability to use a common configuration for both the
Lambda handler and a develop server.

```js
// app.js

import express from "express";
import bodyParser from "body-parser";
import cors from "cors";

// Notice how this function is asynchronous! This is the ideal place
// to initialize your database or any other services your app might need.
export default async function app() {
  const expressApp = express();
  expressApp.use(cors()); // Enables CORS headers
  expressApp.use(bodyParser.json());

  expressApp.get("/hello", (req, res) => {
    res.send("Hello world!");
  });

  return expressApp;
}
```

Next, a custom server that we can use for development. This provides a more
typical development experience than emulating Lambda locally. You can also use
this anywhere you need a server.

```js
// main.js

import app from "./app";
import http from "http";

const PORT = Number(process.env.PORT || 4000);
const HOST = process.env.HOST || "localhost";

async function bootstrap() {
  const expressApp = await app();
  const server = http.createServer(expressApp);

  server.listen(PORT, HOST, () => {
    const { address, port } = server.address();
    console.log(`Server is running at http://${address}:${port}! 👾`);
  });
}

bootstrap();
```

Lastly, the start of the show: a Lambda handler that uses our Express instance.
This is what we are going to deploy to the cloud.

```js
// lambda.js

import app from "./app";
import serverlessExpress from "@codegenie/serverless-express";
import "source-map-support/register"; // Enables support for source maps!

// We don't need to recreate the entire application if our Lambda
// is already up and running when we receive a new request.
// That's why we cache its instance here.
let serverlessExpressApp;

export async function handler(event, context) {
  if (!serverlessExpressApp) {
    const expressApp = await app();
    serverlessExpressApp = serverlessExpress({ app: expressApp });
  }

  return serverlessExpressApp(event, context);
}
```

## Run, build and deploy

[`@vercel/ncc`](https://www.npmjs.com/package/@vercel/ncc) is a lovely utility
that takes care of the complexity of the building process because it also
handles [native modules](https://nodejs.org/api/addons.html) and minification.

Having a small bundle is ideal for container-based applications because it
takes less times to load into memory, reducing cold boots, and consequentially,
costs.

Setting it up is pretty easy:

```jsonc
{
  "main": "src/main",
  "scripts": {
    "start": "ncc run",
    // `-m` flag minifies the code!
    "build": "ncc build -m src/lambda",
  },
  // ...
}
```

And this is how you deploy the code:

```sh
npm run build
(cd dist/ && zip -r - ./) >serverless-express-example.zip
# Replace `serverless-express-lambda` with the name of your Lambda function
aws lambda update-function-code \
  --function-name serverless-express-example \
  --zip-file fileb://serverless-express-example.zip
```

You can view the full code sample on GitHub
[here](https://github.com/th3rius/serverless-express-starter). Cheers!
