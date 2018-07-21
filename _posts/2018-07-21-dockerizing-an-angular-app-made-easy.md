---
layout: post
title: Dockerizing an Angular app made easy
tags:
    - Open source
    - Tutorial
    - DevOps
---

Ok, you made an Angular app and you want to dockerize it painless. Join the dark side...
<!--more-->

First of all, don't be scared, it's not as hard as it could seem (maybe). 

I will use a project in which I collaborated on GitHub to explain how you can dockerize you Angular app without too many
issues. It's simple once you understand the logic. Trust me, I'm an engineer!

Let's take a look at the original project. It's called **Open home**, and of course you can see the full project [here](https://github.com/open-home)
on GitHub (did I say that I love open source?).

Now that you can see the source, let's talk about this app. 

We will use the *openhome-panel* repo, which contains the monitor panel for the services involved (more on the README file).

If you look closer, you will notice a *Dockerfile* in the root of the project. Let's start digging deeper...

```bash
### STAGE 1: Build ###

FROM node:8.11.1-alpine as builder

# Preparing working environment.
RUN mkdir -p /usr/src/openhome-panel
WORKDIR /usr/src/openhome-panel
```

Ok, what are we doing? 

To build your app and your custom image for the future container, you will need an original image first.
So we're getting `node:8.11.1-alpine` as base image to have a Node.js runtime to run our Angular application.
Once we download it, we will create a folder inside it to push our code in with the `RUN` statement, and set it as our working directory.

I suppose you have already noticed that `STAGE 1` comment and that `as builder` after the name of the base image.

This is one of greatest features of Docker, the multi-stage building. Essentially, we can setup multiple stages and keep up track
of our steps in a single Dockerfile, allowing us to have a clearer building process and use some tricks in some cases.

For example, `as builder` means that we will refer to the first image step simply by calling it "builder", as we will see later.

```bash
# Installing dependencies.
COPY package*.json /usr/src/openhome-panel/
RUN npm install

# Copy openhome-panel source into image.
COPY . /usr/src/openhome-panel

# Building app.
RUN npm run-script build
```

Now it's time to prepare the right environment. We will inject our `package.json` file into the image, so we can install the 
dependencies, copy our source code and use npm again to build the Angular code. Yes, we just copied the source from our 
local machine into a temporary image that technically speaking, it's not really existing. Cool right? Let's move ahead, 
we're closer than what you could probably think.

```bash
### STAGE 2: Setup ###

FROM nginx:1.13.12-alpine

# Removing nginx default page.
RUN rm -rf /usr/share/nginx/html/*

# Copying nginx configuration.
COPY /nginx/nginx.conf /etc/nginx/conf.d/default.conf
```

Stage 2, yeah! To run our application we need a server, of course! (Don't you say?!)

I choose `nginx:1.13.12-alpine` because it's a very small image and works really well for what we're doing, but you can choose
the server you are most comfortable with obviously.

Essentially, we are removing the nginx default page inside the image and injecting our custom configuration file to make it
work as we want. Let's take a look at it:

```bash
server {
  listen 80;
  location / {
    root /usr/share/nginx/html;
    index index.html index.htm;
    try_files $uri $uri/ /index.html =404;
  }
}
```

Very simple, nothing special. C'mon, we are almost done, let's go.

```bash
# Copying openhome-panel source into web server root.
COPY --from=builder /usr/src/openhome-panel/public /usr/share/nginx/html

# Exposing ports.
EXPOSE 80

# Starting server.
CMD ["nginx", "-g", "daemon off;"]
```

Here the magic happens in multistage builds. We copy the built app code from the previous stage with the `COPY` statement
from the `builder`, we inject it in this stage of the image and we expose the default 80 port. It's magic!

Our image is ready, but we still have to start the server! At the end of every Dockerfile you will need to specify the default
first command in order the get the server, or whatever service it is, running. This is what `CMD` statement is made for 
and the last line makes our image become a fully working container ready to serve our Angular application.

But we are not done, yet. Now it's time to get the container up and running!

If your have successfully built the image with `docker build -t image-name .`, now we have to fire this container up! Ready?

`docker run -d -p 80:80 --name container-name image-name`

If everything's ok, just visit your *localhost* (or whatever address based on your configuration) with a browser to see
your beautiful dockerized app running and shining inside a container! You see? It's running! Done!

Oh, but now don't forget your friends. You can visit my GitHub profile by following the link at you left, but take a look at
[Openhome](https://github.com/open-home) project again and most importantly [here](https://github.com/ukaoskid), to see the 
full profile of **Simone Di Cicco**, the guy who's behind Openhome.

C'mon, you didn't expect it to be so easy.