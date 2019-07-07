---
title: "Journey of an Ops guy from Monolithic to Microservice"
date: 2019-07-07T07:07:07+05:30
draft: false
---
<details>
 <summary>This is story of my journey at [Synup](https://synup.com), an awesome little startup (Now it’s getting big!) where i learned tremendously and i guess i truly transformed from Ops guy to a DevOps.</summary>
 This blogpost talks about how to handle Ops stack in era of Microservices, just scratchs the surface.
</details>
<!--more-->

> When these transformations started in [Synup](https://synup.com), i was only DevOps/Ops guy so i got my hands dirty in almost everything. Note: I merely handle the Ops part, i am not expert on anything else and did not handle any other part like business logic. My Manager [Akash](https://hashnuke.com) handled the all things as director of Engineering at [Synup](https://synup.com). He formulated all plans of how to carry out these transformations and helped me a lot.

In this blog, i want to explore what happened in this journey of Monolithic to Microservice through Ops perceptive (as you guessed!, i am an Ops guy). We will start our journey with why we want microservice and then will look at how Ops stack changed from this decision. I want to keep blog post as much as short i can. For deep i will open-sourcing some tools that i made along the way.
So without further a due, let start…

**Why Microservice?**

I am not going to argue here that why it’s bad or good but all it boils down to scale (according to me 😜 ).  When i say scale, there are two things. first scaling of your application/product to serve more customer and second is increasing team size working on it so more feature can be added fast. And to solve both of these scaling, you somehow move to Microservice architecture. (If you wanna scale your product or application, you need to break big applications(Monolithic) into individual scalable services (Microservice) so that it can scale easily and also smooth the burden of maintaining or adding a feature to it.)

Now let’s say that you started said journey, what you done, somehow you will create many individual services/projects with clear responsibility. Naturally it will create many codebases and now you need to make decision how you want to organise these codebases, something like monorepo or individual repo.
We went with monorepo for easily collaboration between different codebases like one pulll-request can make changes for different services if required, great idea by [Akash](https://hashnuke.com/). And this decision lead to write some wonderful tooling (more on this later) and easily enforcement of some conventions. I will actually show the example monorepo with the tooling we build that simplified our lives.

**Requirement of new Ops Stack as arised from these microservices**

So let’s say that due to variety of reasons you wanna move to microservices from monolithic, how as an Ops person you gonna handle it. You need to figure out how to host these microservices, not to get lost in it and give developers some consistent environment across these microservices. So due to these things, many old things/tools get thrown out of window like hosting/maintaning these services on individual servers, non-automated infrastructure setup, packaging/distributing these services with each’s language tooling etc.
Seeing these problems, you pretty much realise that you need some kind of magical system like orchestrator that works across above limitations to which you give microservice through well defined structure and it’s just runs it. One such an orchestrator engine is Kubernetes that takes input as bunch of yaml of desired state of services and maintain it. But i am not here to talk about Kubernetes, maybe in another blogs 😆.

### App Configuration in Microservice architecture

But quickly enough i realised that configuration part of microservices has to be aligned with whatever i am planning for Ops stack. Here by configuration i mean values needed by these services to run properly like database URI to which database to connect to and etc. I am big proponent of [12 factor App](https://12factor.net/) and it suggests that configuration values should not in codebase. Simply putting when you make tarball/container-image of application, you should be able to run it on any environment (like from staging to production) with providing different values to these configuration. Secret management is also part of this problem.

As an Ops guy, i was trying to have one unified system for all services and without a consistent Configuration standard for all services, it was very hard goal. Big part of any applicatin configuration system is to provide management of needed values between different environment (from development to staging to production). Development and Production environment are relatively simple as you want to run app with mostly single configuration for each. On development, mostly each service will run locally on developer’s machine and he/she will have complete authority for configuration of it and mostly dependent services are also going to be running locally like database, Redis etc. Production is also simple as all services wants to be run as one global configuration and you probably never wants to run multiple configurations for them, simply putting on production one application will one set of configuration values. But staging environment on the other hand is the beast, here you want to run multiple configurations of services. Let’s say that simultaneously many features is getting worked on for a service and QA team wants to tests these feature at the same time also. This arises the situation of hosting multiple instance of this service with isolation with one other on single staging environment. And you want to define a configuration system that can handle these situations with the breeze. As you can see, this situation needs to handled by both sides from Ops and dev.

At [Synup](https://synup.com), we decided to use consul's key-value store for config management (as we were already using Consul for service discovery and distributed locks). Choosing Consul did solve the problem of config being hardcoded in application and secret management but did not solve the above described problem and problem of consistency. So we decided to write a wrapper library for each language on top of Consul API for these. I suggested to use directory structure for solving of the problem of staging and implement these standards in Consul wrapper library so that services did not worry about internal details. We decided to use following directory for service configuration home.
```
/SERVICE_NAME/ENVIRONMENT/SOME_RANDOM_STRING
```
like: `/app/staging/main`.
For more more information look at **https://github.com/rahulwa/ji/tree/master/consul_pyconfig**, it is wrapper library in python for facilitating these implementations and supports overwrite values with environment variables.

### Deployment in Microservice architecture

Deployment can vastly be simplified/standardised for microservices using Kubernetes. Kubernetes can be daunting and it is not a PaaS (platform as a service) solution out of the box.
You need to setup lot of things before you can start hosting production application on it like ingress setup, how you gonna deploy application, logging for application, monitoring of application/cluster and etc (though we are not going to talk about these things except deployment).

In [Synup](https://synup.com), we adapted [helm chart](https://helm.sh/) for deployment config, but why? As you start writing deployment for application in [plain yaml](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/#creating-a-deployment), pretty soon you will realise need for templating engine that can do variable assignment, conditions, looping and etc. Helm solves these problems and also gives [standard structure](https://helm.sh/docs/developing_charts/#the-chart-file-structure) to deployment package, about how they should be organised.
We created a `charts` subfolder in every service and put deployment config there, it gave us git committed deployment scripts and which is bundled with application (awesome, right!!). It’s great but it doesn’t give you one command for deployment, still you need to make image of your application and push it somewhere and then trigger `helm upgrade`  with these config. Here comes a simple tool that does specifically this task, [skaffold](https://github.com/GoogleContainerTools/skaffold), basically it build docker image of your application then does `kubectl apply` /`helm upgrade` to deploy it. So you deployment command becomes one command:  `skaffold run -f skaffold-production.yml`.

Great, we got one command for deployment but it doesn’t provide seemless experience as for checking logs we need to run different set of commands, for restarting/config-reload of application we need to do completely different set of commands etc. I did not see this problem but my manager [Akash](https://hashnuke.com) pointed it out and suggested to write a script that gives a simple commands for doing basic stuffs. It generated a tool called `ji` which is simple bash script organised using [sub](https://github.com/basecamp/sub). Monorepo style helps us to achieve this without much hassle as ji and other services code live in same repo so ji knows about the location of these codebases with simple relative path. General command looks like:
```
ji <action> <environment> <services> <prefix_string>
```
where *action* can be up/down/log/logs/restart/reload, *environment* can be production/staging/local and *prefix_string* can be any string. let's see it in action, run it full screen as terminal is getting truncated. <script id="asciicast-255857" src="https://asciinema.org/a/255857.js" async></script>
Now let’s talk about what basic tasks it supports and how.
```sh
ji up staging foo main
```
which is going to deploy `foo` service on staging with a name `main-foo`, here it can be easily deploy second instance of same application on staging as `ji up staging foo other`. `ji up` is for deploying the application. It `cd` into service’s charts directory and runs `skaffold-<environments>.yaml` file with some validation and exports `COMPONENT_PREFIX` with value of `<prefix_string>` needed by helm chart to decide which key-value folder on Consul to connect to get config for this service. So config directory on Consul `/SERVICE_NAME/ENVIRONMENT/<SOME_RANDOM_STRING>`'s `SOME_RANDOM_STRING` becomes `prefix_string` on ji deployment.
```sh
ji attach staging foo main
```
attaches to container's terminal running though dialog selection for `main-foo` helm application.
```sh
ji logs staging foo main
```
tails the logs of all pods for all k8s deployments with `main-foo` name prefix.
```sh
ji log staging foo main
```
tails the logs of all pods for a particular k8s deployment through a selection menu for `main-foo` helm application.
```sh
ji restart staging foo main
```
is going to kill all pods with name prefix `main-foo` and Kubernetes will spawn new ones in-place of these, like restart. It’s hard restart.
```sh
ji reload staging foo main
```
is going to update all k8s deployment objects with a random environment variable and it will trigger rolling deployment of existing deployment. It’s a soft restart in the sense that application will not be down at any particular point of time.
```sh
ji info staging foo main
```
will print the current status of deployed foo application with main prefix (`main-foo` helm application).
```sh
ji down staging foo main
```
will uninstall `main-foo` helm application.

For more info, you can look at **https://github.com/rahulwa/ji/tree/master/ji**.
