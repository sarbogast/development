# Continuous Deployment of Couchbase Mobile applications with Docker Compose, Docker Hub and Tutum

The Couchbase Mobile stack in it's simplest form consists of 3 components that are shown in the diagram below:

![](assets/couchbase-mobile-stack.png)

**Couchbase Lite** is a NoSQL mobile database that persists data in JSON and binary format.
**Sync Gateway** is the middle-tier server that exposes a database API for Couchbase Lite databases to replicate to and from (data is not persisted in Sync Gateway).
**Couchbase Server** is a NoSQL server that's used as a storage engine by Sync Gateway.

**NOTE:** You could use Couchbase Lite as a embedded database only with the sync capabilities but for this tutorial we will consider the most common use cases.

In this stack, each component has a clear responsibility and can greatly reduce the time it takes to build a suit of applications (Android, iOS, Web...) without compromising on the user experience because the user's data is automatically synched to Couchbase Lite in the background.

From a deployment standpoint (especially if you are not a DevOpsy person!) it may look like a nightmare but thanks to the Docker Toolbox and Tutum (now part of Docker!), you can continuously deploy each component individually with a simple **git push**. Here are the core concepts that you will learn in this tutorial:

- Basic Docker commands for Sync Gateway
- The development environment with Docker Compose and code sharing on GitHub
- Adding more components (here, a Web App) in the lifecycle of the project
- Setting up the Docker Hub repositories
- Continuously Deploying with Tutum

By the end of the tutorial, the release piple will look like this:

![](assets/pipeline.png)

And your project directory will have the following structure:

// img

## Getting Started with Docker

If you are new to Docker, be sure to follow [this guide](https://docs.docker.com/mac/step_one/) to install the Docker Toolbox. It will install a number of other tools such as Docker Machine and Docker Compose that we will need later. Use the **Docker Quickstart Terminal** from the Launchpad on Mac OS X, this will start a new VM with Virtual Box, configure it as a Docker Host and set up the Docker client to connect to it in a new Terminal window:

![](assets/launchpad.png)

Check the Terminal window for the IP address of the Docker host as you will need it later. Run `docker -v` and `docker-compose` to make sure the binaries were installed correctly.

If you try to run `docker ps` (the command to list all containers) nothing happens and that's because you need to tell Docker to connect to the Docker machine you created above. Docker uses environment variables for this, run `eval "$(docker-machine env default)"` and then run `docker ps` again which should work this time. 

The great thing about using Docker in your development environment is that you don't need to install your application dependencies on your host machine. They can reside in a Docker container running the application. Declaring the dependencies to be used in a Docker container is done in the Dockerfile and luckily for us there are official repositories on Docker Hub for [Sync Gateway](https://hub.docker.com/r/couchbase/sync-gateway/) and [Couchbase Server](https://hub.docker.com/r/couchbase/server).

In the same Terminal window, create a new directory called **sync-gateway-docker** and inside that folder create a new **sync-gateway-config.json** file with following:

```javascript
{
     "log":["*"],
     "verbose": true,
     "databases": {
          "kitchen-sync": {
             "server":"walrus:",
             "users": {"GUEST": {"disabled": false, "all_channels": ["*"], "admin_channels": ["*"]}},
             "sync":`function(doc) {channel(doc.channels);}`
          }
     }
}
```

You're creating a database called **kitchen-sync** and using the **Walrus** mode which means that all the documents are stored in memory. This is especially convenient in development when you are testing the functionalities of your app and don't need a Couchbase Server running to persist data. In the directory where you created the config file, start Sync Gateway in a Docker container:

```bash
$ docker run -v '/Users/jamesnocentini/Developer/mini-hacks/continuous-deployment/:/www/' -p 4984:4984 couchbase/sync-gateway /www/sync-gateway-config.json
```

Here's what is happening:

- **v**: The v flag stands for volume. You're mounting the current directory of the host which contains the config file in a new **www** directory in the container.
- **p**: You're mapping port 4984 of the container to the same value in the docker host (your machine).
- **couchbase/sync-gateway**: The name of the image hosted on Docker Hub to run (see official [couchbase/sync-gateway](https://hub.docker.com/r/couchbase/sync-gateway/) page).
- **/www/sync-gateway-config.json**: The path to the config file you mounted in the container.

Now open a new browser window at **http://192.168.99.100:4984** (replace the IP address if yours is different). You should see the Sync Gateway welcome message:

![](assets/browser-user-port.png)

Hooray! You've got your first Sync Gateway container running. In the next section, we'll introduce a few more components (Couchbase Server and a Web App) that will each run in different containers. But first, you need to grab some source code so fire up your Git commands.

## Code sharing on GitHub

To keep it simple, we have already published the code of the different components under the **Kitchen-Sync** GitHub organisation. There are 3 repositories:

- [sync-gateway]() contains the configuration file to pass to Sync Gateway instances.
- [web]() is a simple Web App built with ReactJS on the front-end and a Node.js web server that connects to the Sync Gateway REST API.
- [development]() ties all of the components together in the same directory as submodules. There is also a `README.md` and `docker-compose.yml` file that will help us start all the different components very easily.

In a new directory, run the following commands:

```bash
$ git clone git@github.com:Kitchen-Sync/development.git Kitchen-Sync
$ cd Kitchen-Sync
$ git submodule init
$ git submodule update
```

### How-To

To do the same for your project, head over to the [New Organization](https://github.com/organizations/new) page and create the different repositories as needed in your application. Don't forget to create the entrypoint repository to other applications that will be added as submodules.

```bash
$ git clone git@github.com:<project>/development.git <project>
$ cd <project>
```

Next you can add each application repository as a submodule:

```bash
$ git submodule add git@github.com:<project>/sync-gateway-config.git
$ git submodule add git@github.com:<project>/web.git
```

Now, if you commit and push to GitHub, the development repository will reference the submodules as well:

![](assets/submodules.png)

## Docker Compose for development

Now that you have the source code for the Web App and Sync Gateway configuration file nicely organized you start using Docker Compose to orchestrate and manage different Docker containers. Open `docker-compose.yml` and let's have a look:

```bash
web:
  build: ./web
  ports:
    - '3000:3000'
  links:
    - syncgateway
syncgateway:
  image: 'couchbase/sync-gateway:latest'
  command: /www/dev.json
  volumes:
    - ./sync-gateway-config/:/www/
  ports:
    - '4984:4984'
```

The first container image is called **web** and is built directly out of the **web** directory. That's because there is a **Dockerfile** inside of that directory to that describes the steps to build the image. We won't cover [Dockerizing applications]() in this tutorial but it's crucial that you understand how because unless you use an image that is already published to Docker Hub, you will need to Dockerize your application components to get Continuous Delivery set up correctly. The second container is called **syncgateway** and is using the official [couchbase/sync-gateway]() image with the development config file. The options specified are the same as the one you specified in the `docker run` in the first section of this tutorial.

Run this command to start both containers:

```bash
$ docker-compose up
```

Notice how the logs from Sync Gateway (in blue) and from the Web App (in yellow) are aggregated. This is huge win for productivity during development as you don't have to switch between different Terminal tabs.

![](assets/logs.png)

Next, open **http://192.168.99.100:3000** (replace the IP address if yours is different) and start adding items. Items are persisted to Sync Gateway, reloading the page will fetch the documents from Sync Gateway:

![](assets/development-web-persist.png)

Well done! You've decoupled the Sync Gateway configuration file from the Web App in the source code but the development experience remains streamlined and simple with Docker Compose.

## Docker Hub

**NOTE:** To reproduce the steps below with your own Docker Hub and Tutum accounts you will need to be a member of the [Kitchen-Sync](https://github.com/Kitchen-Sync). Ping me your GitHub username on [Twitter](http://twitter.com/jamiltz) or try the following with your own applications components.

Remember how `docker-compose.yml` builds the image for the container named **web** from the local directory? Well, that's perfect during development because you can change the source and simply run `docker-compose up` again. For production deployments however, we need to have a shared image that can be pulled by services such as Tutum. Docker Hub is the perfect tool to host container images online so head over to [hub.docker.com](http://hub.docker.com) and do the following:

1. Click the **Organizations** tab and create a new organization name **KitchenSync** (you may have to append characters to ensure uniquness of the name as it will already exist on Docker Hub)

Create a new repository with the following steps:

1. Click **Add Repository** > **Automated Build**
2. You will be prompted to link your GitHub account and after doing so you will see the list of organizations and repositories
3. Select the repository named **web** from the **Kitchen-Sync** GitHub organization (only visible if you are a member of the **Kitchen-Sync** organization) and the following page will appear:

![](assets/automated-build.png)

1. Make sure to choose the organization name from the dropdown menu
2. Check "When active we will build when new pushes occur"
3. Click **Create Repository**

You checked the option in step 3 so you can trigger a new build when a developer from the team has pushed new commits on the **master** branch. The new repository can be found at [https://hub.docker.com/r/kitchensync/web/](https://hub.docker.com/r/kitchensync/web/).

Next, you can push the source code of your application component:

![](assets/git-push.png) 

And see the action in the **Build Details** tab of the Docker Hub repository:

![](assets/build-details.png)

## Tutum

For this section, you're going to use Tutum.

### Creating a node cluster ready for Docker deployment

1. Create an account on Tutum and sign in
2. Got to **Account info** > **Cloud providers** and connect the providers of your choice.
3. Under the **Nodes** tab, click **Launch new node cluster**
4. Select a cluster name (e.g. kitchen-sync-staging), server type and cluster size (one should be enough for this example) then click **Launch node cluster**

![](assets/staging-node.png)

At this point, you should have your server(s) ready for production and reachable on a node.tutum.io sub-domain (in my case it was .

## Deploying the application stack

Time to use a production-ready version of the application: the Docker image built by Docker Hub. Remember I told you the `docker-compose.yml` will come handy? To deploy our application, we’re gonna use Tutum Stacks, which are YAML files very similar to Docker Compose files.
Here's the `tutum-staging.yml` we’re going to use:

```
web:
  image: kitchensync/web
  ports:
    - '3000:3000'
  links:
    - syncgateway
  tags:
    - kitchen-sync-staging
syncgateway:
  image: 'couchbase/sync-gateway:latest'
  command: 'http://git.io/vWnCH'
  ports:
    - '4984:4984'
  tags:
    - kitchen-sync-staging
```

Here, you're specifying a URL pointing to the Sync Gateway configuration file. It's in fact the raw URL of the `development.json` configuration file that was shortened with [http://git.io](git.io).

Now we have out `tutum-staging.yml` and Web App image on Docker Hub, let's deploy:

1. Under the **Stacks** tab, click on **Create your first stack**
2. Set a name for your stack (e.g kitchen-sync-staging)
3. Paste the content of your **tutumstaging.yml** in the Stack file textarea
4. Click on **Create and deploy**

Wait for both services to be up:

![](assets/service-dashboard.png)

And open your tutum node subdomain on port 3000:

![](assets/web-app-staging.png)

**Congratulations!** You just deployed your application.

## Continuously deploy your application

We are a few extra steps from achieving continuous deployment. We now want to automatically redeploy our application service whenever there is a new image available on Docker Hub. The same way we triggered the Docker Hub build from CircleCI, we are going to trigger a Tutum Redeploy from Docker Hub when a new image is successfully published.

1. From the **Services** tab, click on your application service
2. Go to the **Triggers** tab and add a redeploy trigger named Docker Hub
3. Refresh the page (this is needed at the time of writing) and copy the trigger URL
4. Go back to your Docker Hub repository page, then click on **Settings > Webhooks**
5. Add a webhook calling the trigger URL you just copied

Now, push some edits to your repository and follow your application as it goes through your deployment pipeline:

Development→ GitHub → CircleCI → Docker Hub → Tutum → Production