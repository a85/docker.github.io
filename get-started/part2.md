---
title: "Get Started, Part 2: Containers"
keywords: containers, python, code, coding, build, push, run
description: Learn how to write, build, and run a simple app -- the Docker way.
---

{% include_relative nav.html selected="2" %}

## Prerequisites

- [Install Docker version 1.13 or higher](/engine/installation/).
- Read the orientation in [Part 1](index.md).
- Give your environment a quick test run to make sure you're all set up:

  ```
  docker run hello-world
  ```

## Introduction

It's time to begin building an app the Docker way. We'll start at the bottom of
the hierarchy of such an app, which is a container, which we cover on this page.
Above this level is a service, which defines how containers behave in
production, covered in [Part 3](part3.md). Finally, at the top level is the
stack, defining the interactions of all the services, covered in
[Part 5](part5.md).

- Stack
- Services
- **Container** (you are here)

## Your new development environment

In the past, if you were to start writing a Python app, your first
order of business was to install a Python runtime onto your machine. But,
that creates a situation where the environment on your machine has to be just
so in order for your app to run as expected; ditto for the server that runs
your app.

With Docker, you can just grab a portable Python runtime as an image, no
installation necessary. Then, your build can include the base Python image
right alongside your app code, ensuring that your app, its dependencies, and the
runtime, all travel together.

These portable images are defined by something called a `Dockerfile`.

## Define a container with a `Dockerfile`

`Dockerfile` will define what goes on in the environment inside your
container. Access to resources like networking interfaces and disk drives is
virtualized inside this environment, which is isolated from the rest of your
system, so you have to map ports to the outside world, and
be specific about what files you want to "copy in" to that environment. However,
after doing that, you can expect that the build of your app defined in this
`Dockerfile` will behave exactly the same wherever it runs.

### `Dockerfile`

Create an empty directory and put this file in it, with the name `Dockerfile`.
Take note of the comments that explain each statement.

```conf
# Use an official Python runtime as a base image
FROM python:2.7-slim

# Set the working directory to /app
WORKDIR /app

# Copy the current directory contents into the container at /app
ADD . /app

# Install any needed packages specified in requirements.txt
RUN pip install -r requirements.txt

# Make port 80 available to the world outside this container
EXPOSE 80

# Define environment variable
ENV NAME World

# Run app.py when the container launches
CMD ["python", "app.py"]
```

This `Dockerfile` refers to a couple of things we haven't created yet, namely
`app.py` and `requirements.txt`. Let's get those in place next.

## The app itself

Grab these two files and place them in the same folder as `Dockerfile`.
This completes our app, which as you can see is quite simple. When the above
`Dockerfile` is built into an image, `app.py` and `requirements.txt` will be
present because of that `Dockerfile`'s `ADD` command, and the output from
`app.py` will be accessible over HTTP thanks to the `EXPOSE` command.

### `requirements.txt`

```
Flask
Redis
```

### `app.py`

```python
from flask import Flask
from redis import Redis, RedisError
import os
import socket

# Connect to Redis
redis = Redis(host="redis", db=0)

app = Flask(__name__)

@app.route("/")
def hello():
    try:
        visits = redis.incr('counter')
    except RedisError:
        visits = "<i>cannot connect to Redis, counter disabled</i>"

    html = "<h3>Hello {name}!</h3>" \
           "<b>Hostname:</b> {hostname}<br/>" \
           "<b>Visits:</b> {visits}"
    return html.format(name=os.getenv('NAME', "world"), hostname=socket.gethostname(), visits=visits)

if __name__ == "__main__":
	app.run(host='0.0.0.0', port=80)
```

Now we see that `pip install requirements.txt` installs the Flask and Redis
libraries for Python, and the app prints the environment variable `NAME`, as
well as the output of a call to `socket.gethostname()`. Finally, because Redis
isn't running (as we've only installed the Python library, and not Redis
itself), we should expect that the attempt to use it here will fail and produce
the error message.

> *Note*: Accessing the name of the host when inside a container retrieves the
container ID, which is like the process ID for a running executable.

## Build the App

That's it! You don't need Python or anything in `requirements.txt` on your
system, nor will building or running this image install them on your system. It
doesn't seem like you've really set up an environment with Python and Flask, but
you have.

Here's what `ls` should show:

```shell
$ ls
Dockerfile		app.py			requirements.txt
```

Now run the build command. This creates a Docker image, which we're going to
tag using `-t` so it has a friendly name.

```shell
docker build -t friendlyhello .
```

Where is your built image? It's in your machine's local Docker image registry:

```shell
$ docker images

REPOSITORY            TAG                 IMAGE ID
friendlyhello         latest              326387cea398
```

## Run the app

Run the app, mapping your machine's port 4000 to the container's `EXPOSE`d port 80
using `-p`:

```shell
docker run -p 4000:80 friendlyhello
```

You should see a notice that Python is serving your app at `http://0.0.0.0:80`.
But that message coming from inside the container, which doesn't know you mapped
port 80 of that container to 4000, making the correct URL
`http://localhost:4000`. Go there, and you'll see the "Hello World" text, the
container ID, and the Redis error message.

> **Note**: This port remapping of `4000:80` is to demonstrate the difference
between what you `EXPOSE` within the `Dockerfile`, and what you `publish` using
`docker run -p`. In later steps, we'll just map port 80 on the host to port 80
in the container and use `http://localhost`.

Hit `CTRL+C` in your terminal to quit.

Now let's run the app in the background, in detached mode:

```shell
docker run -d -p 4000:80 friendlyhello
```

You get the long container ID for your app and then are kicked back to your
terminal. Your container is running in the background. You can also see the
abbreviated container ID with `docker ps` (and both work interchangeably when
running commands):

```shell
$ docker ps
CONTAINER ID        IMAGE               COMMAND             CREATED
1fa4ab2cf395        friendlyhello       "python app.py"     28 seconds ago
```

You'll see that `CONTAINER ID` matches what's on `http://localhost:4000`.

Now use `docker stop` to end the process, using the `CONTAINER ID`, like so:

```shell
docker stop 1fa4ab2cf395
```

## Share your image

To demonstrate the portability of what we just created, let's upload our build
and run it somewhere else. After all, you'll need to learn how to push to
registries to make deployment of containers actually happen.

A registry is a collection of repositories, and a repository is a collection of
images -- sort of like a GitHub repository, except the code is already built. An
account on a registry can create many repositories. The `docker` CLI is
preconfigured to use Docker's public registry by default.

> **Note**: We'll be using Docker's public registry here just because it's free
and pre-configured, but there are many public ones to choose from, and you can
even set up your own private registry using [Docker Trusted
Registry](/datacenter/dtr/2.2/guides/).

If you don't have a Docker account, sign up for one at
[cloud.docker.com](https://cloud.docker.com/). Make note of your username.

Log in your local machine.

```shell
docker login
```

Now, publish your image. The notation for associating a local image with a
repository on a registry, is `username/repository:tag`. The `:tag` is optional,
but recommended; it's the mechanism that registries use to give Docker images a
version. So, putting all that together, enter your username, and repo
and tag names, so your existing image will upload to your desired destination:

```shell
docker tag friendlyhello username/repository:tag
```

Upload your tagged image:

```shell
docker push username/repository:tag
```

Once complete, the results of this upload are publicly available. From now on,
you can use `docker run` and run your app on any machine with this command:

```shell
docker run -p 4000:80 username/repository:tag
```

> Note: If you don't specify the `:tag` portion of these commands,
  the tag of `:latest` will be assumed, both when you build and when you run
  images.

No matter where `docker run` executes, it pulls your image, along with Python
and all the dependencies from `requirements.txt`, and runs your code. It all
travels together in a neat little package, and the host machine doesn't have to
install anything but Docker to run it.

## Conclusion of part two

That's all for this page. In the next section, we will learn how to scale our
application by running this container in a **service**.

[Continue to Part 3 >>](part3.md){: class="button outline-btn" style="margin-bottom: 30px"}


## Recap and cheat sheet (optional)

Here's [a terminal recording of what was covered on this page](https://asciinema.org/a/blkah0l4ds33tbe06y4vkme6g):

<script type="text/javascript" src="https://asciinema.org/a/blkah0l4ds33tbe06y4vkme6g.js" id="asciicast-blkah0l4ds33tbe06y4vkme6g" speed="2" async></script>

Here is a list of the basic commands from this page, and some related ones if
you'd like to explore a bit before moving on.

```shell
docker build -t friendlyname .  # Create image using this directory's Dockerfile
docker run -p 4000:80 friendlyname  # Run "friendlyname" mapping port 4000 to 80
docker run -d -p 4000:80 friendlyname         # Same thing, but in detached mode
docker ps                                 # See a list of all running containers
docker stop <hash>                     # Gracefully stop the specified container
docker ps -a           # See a list of all containers, even the ones not running
docker kill <hash>                   # Force shutdown of the specified container
docker rm <hash>              # Remove the specified container from this machine
docker rm $(docker ps -a -q)           # Remove all containers from this machine
docker images -a                               # Show all images on this machine
docker rmi <imagename>            # Remove the specified image from this machine
docker rmi $(docker images -q)             # Remove all images from this machine
docker login             # Log in this CLI session using your Docker credentials
docker tag <image> username/repository:tag  # Tag <image> for upload to registry
docker push username/repository:tag            # Upload tagged image to registry
docker run username/repository:tag                   # Run image from a registry
```
