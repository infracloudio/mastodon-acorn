# Deploying Mastodon on Acorn

[Mastodon](https://joinmastodon.org/) is a free and open-source software for running self-hosted social networking services. It has microblogging features similar to Twitter, which are offered by a large number of independently run nodes, known as instances or servers, each with its own code of conduct, terms of service, privacy policy, privacy options, and content moderation policies.

[Acorn](http://www.acorn.io), a user-friendly cloud computing platform, simplifies deploying modern cloud-native apps with a free sandbox accessible through GitHub. It streamlines development workflows using mainstream container tools, providing the power of Kubernetes and Terraform without complexity. To deploy, define your application with an [Acornfile](https://docs.acorn.io/reference/acornfile), generating a deployable Acorn Image. This tutorial illustrates provisioning a Mastodon server on Acorn.

If you want to skip to the end, just click [Run in Acorn](https://acorn.io/run/ghcr.io/infracloudio/mastodon-acorn:v4.%23.%23-%23?ref=slayer321&name=mastodon)(Click on `Customize before deploying` and provide the SMTP details that is required) to launch the app immediately in a free sandbox environment. All you need to join is a GitHub ID to create an account.

If you want to follow along, I’ll walk through the steps to deploy Mastodon Server using Acorn.

_Note: Everything shown in this tutorial can be found in [this repository](https://github.com/infracloudio/mastodon-acorn)_.

## Pre-requisites

- Acorn CLI: The CLI allows you to interact with the Acorn Runtime as well as Acorn to deploy and manage your applications. Refer to the [Installation documentation](https://docs.acorn.io/installation/installing) to install Acorn CLI for your environment.
- A GitHub account is required to sign up and use the Acorn Platform.

## Acorn Login
Log in to the [Acorn Platform](http://beta.acorn.io) using the GitHub Sign-In option with your GitHub user.
![](./assets/acorn-login-page.png)

After the installation of Acorn CLI for your OS, you can login to the Acorn platform.
```
$ acorn login beta.acorn.io
```

## Deploying the Mastodon server
In this tutorial we will deploy Mastodon server.

In the Acorn platform, there are two ways you can try this sample application.
1. Using Acorn platform dashboard.
2. Using CLI

The first way is the easiest one where, in just a few clicks you can deploy the Mastodon on the platform and start using it. However, if you want to customize the application use the second option.

## Deploying Using Acorn Dashboard

In this option you use the published Acorn application image to deploy the Mastodon server in just a few clicks. It allows you to deploy your applications faster without any additional configurations. Let us see below how you can deploy Mastodon server to the Acorn platform dashboard.

1. Login to the [Acorn Platform](https://acorn.io/auth/login)  using the Github Sign-In option with your Github user.
2. Select the “Create Acorn” option.
3. Choose the source for deploying your Acorns
   3.1. Select “From Acorn Image” to deploy the sample Application.
![](./assets/select-from-acorn-image.png)

   3.2. Provide a name "tech-mastodon”, use the default Region and provide the URL for the Acorn image and you need to select "Advanaced Options" and provide all the details required for smtp server.
   ```
   ghcr.io/infracloudio/mastodon-acorn:v4.#.#-#
   ```
![](./assets/mastodon-deploy-preview.png)

_Note: The App will be deployed in the Acorn Sandbox Environment. As the App is provisioned on AcornPlatform in the sandbox environment it will only be available for 2 hrs and after that it will be shutdown. Upgrade to a pro account to keep it running longer_.

4. Once the Acorn is running, you can access it by clicking the Endpoint or the redirect link.
   4.1. Running Application on Acorn
   ![](./assets/mastodon-platform-dashboard.png)
   4.2. Running Mastodon
   ![](./assets/mastodon-homepage.png)


## Deploying Using Acorn CLI
As mentioned previously, running the acorn application using CLI lets you understand the Acornfile. With the CLI option, you can customize the sample app to your requirement or use your Acorn knowledge to run your own Mastodon Server.

To run the application using CLI you first need to clone the source code repository on your machine.

```
$ git clone https://github.com/infracloudio/mastodon-acorn.git
```
Once cloned here’s how the directory structure will look.
```

├── Acornfile
├── conf
│   └── nginx
│       └── mastodon.template
├── LICENSE
├── mastodon.svg
├── README.md
└── tutorial.md
```


### Understanding the Acornfile

We have the Mastodon server ready. Now to run the application we need an Acornfile which describes the whole application without all of the boilerplate of Kubernetes YAML files. The Acorn CLI is used to build, deploy, and operate Acorn on the Acorn cloud platform.  It also can work on any Kubernetes cluster running the open source Acorn Runtime. 

Below is the Acornfile for deploying the Mastodon Server that we created earlier:

```
services: postgres: {
	image: "ghcr.io/acorn-io/postgres:v#.#-#"
}

services: redis: image: "ghcr.io/acorn-io/redis:v#.#.#-#"         


args: {
    smtp_port: "587"
    smtp_server: ""
    ...
    smtp_from_address: "AcornSocial <notification@on-acorn.io>"
}

containers:  {
    web: {
        image: "ghcr.io/mastodon/mastodon:v4.2.0" 
        cmd: ["bash", "-c", "mkdir /mastodon/public/system; bundle exec rake db:setup; rm -f /mastodon/tmp/pids/server.pid; bundle exec rails s -p 3000"]
        ports: publish:"3000:3000/http"
        consumes: ["postgres", "redis"]
        env:{
            LOCAL_DOMAIN: "on-acorn.io"
            WEB_DOMAIN: "@{services.nginx.endpoint}"
            ...
        }
    }
    streaming:{
        image: "ghcr.io/mastodon/mastodon:v4.2.0"
        cmd: ["node","./streaming"]
        consumes: ["postgres", "redis"]
        ports: publish:"4000:4000/http"
        env:{
            LOCAL_DOMAIN: "on-acorn.io"
            WEB_DOMAIN: "@{services.nginx.endpoint}"
            ...
      }
    }
    sidekiq:{
        image: "ghcr.io/mastodon/mastodon:v4.2.0"
        cmd: ["bash", "-c", "bundle exec sidekiq"]
        consumes: ["postgres", "redis"]
        env:{
            LOCAL_DOMAIN: "on-acorn.io"
            WEB_DOMAIN: "@{services.nginx.endpoint}"
            ...
      }
    }
    nginx: {
        image: "nginx"
        ports: publish: "8000:80/http"
        dirs: {
        "/etc/nginx/conf.d": "./conf/nginx"
        }
        env:{
            NGINX_ENDPOINT: "@{services.nginx.endpoint}"
            STREAMING_ENDPOINT: "@{services.streaming.endpoint}"
            WEB_ENDPOINT: "@{services.web.endpoint}"
        }
        cmd: ["/bin/bash", "-c", "envsubst '$${NGINX_ENDPOINT},$${STREAMING_ENDPOINT},$${WEB_ENDPOINT}' < /etc/nginx/conf.d/mastodon.template > /etc/nginx/conf.d/default.conf && nginx -g 'daemon off;'"]
        dependsOn: ["web","streaming","sidekiq"]
    }

}
```

There are different components for running Mastodon server
- web
- sidekiq
- nginx
- db
- redis

The above Acornfile has the following elements:

- **Args**: Which is used to take the user args.
- **Services**:  Here we're using the [Postgres](https://github.com/acorn-io/postgres) and [redis](https://github.com/acorn-io/redis) service that is built into Acorn as an [Acorn Service](https://docs.acorn.io/reference/services).
- **Containers**:  We define different containers with following configurations:
   - **web**: 
       - **image**: using mastodon image
       - **ports**:  port where our web application is listening on.
       - **env**:  In the env section we are providing all the env variables which the application will be using.
       - **consumes**: web consumes postgres and redis
       - **cmd**: command use to run the web component
   - **streaming**: 
       - **image**: using mastodon image
       - **ports**:  port where our streaming application is listening on.
       - **env**:  In the env section we are providing all the env variables which the application will be using.
       - **consumes**: streaming consumes postgres and redis
       - **cmd**: command use to run the streaming component
    - **sidekiq**: 
       - **image**: using mastodon image
       - **ports**:  port where our sidekiq application is listening on.
       - **env**:  In the env section we are providing all the env variables which the application will be using.
       - **consumes**: sidekiq consumes postgres and redis
       - **cmd**: command use to run the streaming component
    - **nginx**: 
       - **image**: using nginx image
       - **ports**:  port where our nginx application is listening on.
       - **env**:  In the env section we are providing all the env variables which the application will be using.
       - **dependsOn**: nginx depends on all the above 3 components
       - **cmd**: command use to run the streaming component

### Running the Application
We have already logged in using Acorn CLI now you can directly deploy applications on your sandbox on the Acorn platform. Run the following command from the root of the directory.

```
$ acorn run -n mastodon . --smtp_login <> --smtp_password <> --smtp_server <>
```

Below is what the output looks like.

![](./assets/mastodon-local-run.png)

## Mastodon Server

Inside the Aconfile all the components are already ready you just need to bring your own SMTP server and provide all the required details.

Once we provide all the details and our Mastodon Server is running below is what our Mastodon dashboard looks like once we logged in as a new user.

![](./assets/mastodon-homepage.png)


_Note: To create the admin account use the `Execute shell` feature on acorn UI for Web component and run below command_

```
RAILS_ENV=production bin/tootctl accounts create \
  alice \
  --email jovon49621@glalen.com \
  --confirmed \
  --role Owner
```

## What's Next?

1. The App is provisioned on Acorn Platform and is available for two hours. Upgrade to Pro account for anything you want to keep running longer.
2. After deploying you can edit the Acorn Application or remove it if no longer needed. Click the Edit option to edit your Acorn's Image. Toggle the Advanced Options switch for additional edit options.
3. Remove the Acorn by selecting the Remove option from your Acorn dashboard.


## Conclusion
In this tutorial we show how we can use the Acornfile and get our Mastodon Server up and running.
Now you can send the URL to your friends and family to join the server.