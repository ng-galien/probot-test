# **DRAFT: Probot CI/CD**

Github App/Action to run CI/CD pipelines in JavaScript.

The purpose of this draft id to describe a simple way to run CI/CD pipelines in JavaScript. It is based on [Probot](https://probot.github.io/), a framework for building Github Apps.  

The motivation is that it's a lot easier to write a CI/CD pipeline in JavaScript than in YAML.  

In addition, pipeline jobs always needs to run shell commands to do something useful. Use shell scripts transform data etc. It's a lot easier to do this in JavaScript than with shell scripts.

Finally, it's a lot easier to write tests for your pipeline instead of live debugging it on your repository.

## **Features**

-   Run CI/CD pipelines in JavaScript
-   Run pipelines on Github Actions
-   Write complex pipeline interactions with NodeRed on the top of Probot.

## Github Action

A Github Action is a way to interact with Github. It can be triggered by a Github event (push, pull request, etc.) or manually. It's typically written in YAML in a `.github/workflows` directory.

```yaml
name: CI

on: [push]

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v1
      - name: Run a one-line script
        run: echo Hello, world!
      - name: Run a multi-line script
        run: |
          echo Add other actions to build,
          echo test, and deploy your project.
``` 


With a Github Action, we run our Javascript pipeline in serverless environment.  

In the user repository there is a pipeline file, for example `.github/pipeline.js`.

```javascript
//Contents of .github/pipeline.js

return context.octokit.issues.createComment(
    context.issue({ body: "Hello, World!" })
);
```

The pipeline file is dynamically loaded and executed by the Github Action. We can use something like [node-eval](https://www.npmjs.com/package/node-eval) package to load and execute the pipeline file.

```javascript

/**
 * @param {import('probot').Probot} app
 */

const nodeEval = require('node-eval');

module.exports = (app) => {
  app.log("Yay! The app was loaded!");

  app.on("issues.opened", async (context) => {
      // Eval the pipeline file
    nodeEval(context, './pipeline.js');
  });
};
```

> The way to run a pipeline in a serverless environment is described [here](https://probot.github.io/docs/deployment/#github-actions).

The serverless environment will be wrapped by the Github Action Adapter. 

> Wrapping into a Github Action is described [here](https://github.com/probot/adapter-github-actions#readme).

## Github App

[Api docs](https://probot.github.io/docs/)

Build a CI/CD application on top of NodeRed, a flow is a pipeline.

### Understanding the Node Red CI/CD flow

A webhooks is coming from the github repository, and handled by the flow. The flow handle the message in a NodeRed style and outputs to a specialized node that will handle the github api.  

### Github application Webhook

> The way to configure the webhook is described here [https://developer.github.com/webhooks/](https://probot.github.io/docs/development/#manually-configuring-a-github-app)


[Smee.io](https://smee.io/) is a proxy that will forward the webhook to the NodeRed application endpoint.  

```bash
npm install --save smee-client
```

```javascript
const SmeeClient = require('smee-client')

const smee = new SmeeClient({
  source: 'https://smee.io/cOVilteS7QGzaXe1',
  target: 'http://localhost:3000/events',
  logger: console
})

const events = smee.start()

// Stop forwarding events
events.close()
```

Probot as a built-in support with setting environment variable `WEBHOOK_PROXY_URL`.

```bash
$ export WEBHOOK_PROXY_URL=https://smee.io/cOVilteS7QGzaXe1
```

### Node Red CI/CD flow

The entry point of the flow is the `HTTP In` node. The node will handle the webhook and output the message to the flow.

![](https://cookbook.nodered.org/images/http/create-an-http-endpoint.png)

The message will we processed by user defined nodes. The output of the flow will be handled at the end by custom function node.

> Documentation for [writing a function node](https://nodered.org/docs/user-guide/writing-functions)

To give an access to the github api we must inject the `context` object used in the Probot API to the global context of node red.

```javascript
// Probot context to inject in the global context of node red
// https://probot.github.io/docs/hello-world/

module.exports = (app) => {
  app.on("issues.opened", async (context) => {
    // `context` extracts information from the event, which can be passed to
    // GitHub API calls. This will return:
    //   { owner: 'yourname', repo: 'yourrepo', number: 123, body: 'Hello World !}
    const params = context.issue({ body: "Hello World!" });

    // Post a comment on the issue
    return context.octokit.issues.createComment(params);
  });
};
```
NodeRed permit to inject the context in its's global context with `functionGlobalContext` in setting.js.

```javascript
functionGlobalContext: {
    osModule:require('os')
}
```

> Documentation for [injecting context](https://nodered.org/docs/user-guide/writing-functions#loading-additional-modules)

### Additional features

We must consider to add the ability to store files to the github repository.
Such way can be done with the NodeRed context store.

> Documentation for [context store](https://nodered.org/docs/user-guide/context#saving-context-data-to-the-file-system)

