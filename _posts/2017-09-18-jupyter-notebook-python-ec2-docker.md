---
layout: post
title: Setting up a Jupyter notebook server on an EC2 instance using Docker
description: A guide to having your own Jupyter notebook server running in a Docker container on a free EC2 instance!
category: tech
excerpt_separator: <!--more-->
comments: true
---

Because I'm a fair hand at maths and even better at being late to parties, I've been meaning lately to play around with some data science. However, I ran into a few dilemmas setting myself up:

* First, the Python environment can be surprisingly messy. Coming from languages with more streamlined package management, I found myself investing more time getting my machine set up than I thought I'd have to.
* I move around a lot. I really want to work on this stuff on the go, from multiple different machines.
* I don't want to mess around setting up the environment a second time.

To solve these problems, after discussing it with a good friend of mine and finding out he had a similar setup himself, I decided to host a Jupyter notebook on a free tier EC2 instance. Thereby being able to access it from anywhere!
<!--more-->

To add to it, I would also set up the Python environment in a Docker container. That way, if I ever wanted to move it to another server or run it locally, it would be an absolute cinch.

The following is guide for us to reproduce this process. Let's get started!

## EC2

The first step is to set up our EC2 instance. Log into [AWS](https://aws.amazon.com/) using your Amazon account. If you don't have an Amazon account, well - no sweat. Just go [sign up](https://www.amazon.com.au/) and pretend to your loved ones like you always had one.

Navigate your way to the EC2 services page and create your new instance. You'll have to pick your Linux distribution (I chose the default Amazon Linux) and your instance type. `t2.micro` is the best-performing type within the free-tier, so I recommend going with that.

Next thing to consider is the security groups.
I added an extra rule allowing connections on port 8888, since that's the port we're going to use for the Jupyter notebook. You can also restrict it to certain IPs if you know where you're going to access the server from.

Awesome! We now have our own EC2 server metaphorically spinning away in the cloud. Now it's time to jump in and set up ourselves up.

Before we move on, make a note of the public DNS of your instance.

## SSH in

You would have been prompted to create a key pair during your EC2 setup process. Since you're a responsible person who follows instructions, you would have also saved it in a secure location. Let's go there now and set the permissions of the `.pem`:

```
cd my_secure_keypair_directory
chmod 400 my-key-pair.pem
```

Now we can ssh into our instance using:

```
ssh -i my-key-pair.pem ec2-user@my.ec2.instance.dns
```

Alternatively, if you're lazy like me, just add it to your ssh-agent with

```
ssh-add my-key-pair.pem
```

## We're in!

Ok, we're finally in. Now it gets interesting.

Update the installed packages:
```
sudo yum update -y
```

Install docker:
```
sudo yum install -y docker
```

Start the docker service:
```
sudo service docker start
```

Add `ec2-user` to the docker group so you can go sudoless:

```
sudo usermod -a -G docker ec2-user
```

You'll have to exit and ssh back in for the user permissions to take effect.

To test that docker was installed correctly, let's try running a Hello World container from the docker training repo:

```
docker run -p 8888:5000 training/webapp
```

Here `-p 8888:5000` is telling docker to link port `5000` from the container to port `8888` on the host, which is the port we opened up in our security groups.

Now if you navigate to `my.ec2.instance.dns:8888`, you should see a friendly `Hello world!`

Well done!


## Jupyter
It's finally time to set up our Jupyter container. For extra credit, consider making your own personal container. Otherwise, if you're a busy person, use a prebuilt one. I'm going to describe setting up [this one](https://hub.docker.com/r/jupyter/datascience-notebook/).

To start with, let's just try and spin it up. Run:

```
docker run -it -p 8888:8888 jupyter/datascience-notebook
```

Take note of the authentication token generated in the server logs, and navigate yourself to your public DNS on port 8888. You should be able to see Jupyter running, and be able to log in with your authentication token and play around.

Nice, right? We've come pretty far, but there's a few problems remaining:

1. I don't want to login using some gibberish token. I want a password.
2. The notebooks I create need to be preserved if the docker container restarts.
3. This container is publicly accessible, so we should really be using HTTPS.

## Password authentication
To spin up the Jupyter server with password authentication, append your command with:
```start-note
book.sh --NotebookApp.password='sha1:f0f9499cc884:3734b043ab7c8e14088cdf6b574d5e9ce6e30fc0'
```
Replace the string in quotes with the your desired password hashed and salted by `IPython.lib.passwd()`.

It's a bit of a catch 22 if you don't have an existing Python setup, but there are online IPython interpreters like [this one](https://www.pythonanywhere.com/try-ipython/) I just googled.

## Notebook persistence

Create a directory to house your notebooks. I created mine as `~/work`.

Now we can run the docker container with the argument `-v ~/work:/home/jovyan/work`. This is setting our host directory `~/work` as the directory to be used for the internal work directory inside docker.

The only issue now is that the docker local user, `joyvan`, doesn't have write permission to our new directory. Easy fix. Let's assign ownership of the folder to a user with uid `1000`:

```
sudo chown 1000 ~/work
```

And now set the docker local user to have a uid of `1000` when starting up the container:

```
NB_UID=1000
```

## SSL

We have two options for SSL. If you like, you can get one signed by certificate authority. A reminder that [Let's Encrypt](https://letsencrypt.org/) offers them for free! Once you have it, simply load it in with the option `--NotebookApp.certfile=/etc/ssl/notebook.pem`.

Alternatively, we can self-sign our own SSL certificate by passing the option `GEN_CERT=yes`. I had to end up adding a security exception to my browser in order to access the site, but since I'm the single intended user, this wasn't really a problem.

# We're done!

Congratulations! We've put all the pieces together. Now the last remaining thing is to run docker as a daemon with the option `-d` and get on with our lives.

Putting it all together, our final docker command is:

```
docker run -dit -v ~/work:/home/jovyan/work -e NB_UID=1000 -e GEN_CERT=yes -p 8888:8888 jupyter/datascience-notebook start-note
book.sh --NotebookApp.password='sha1:f0f9499cc884:3734b043ab7c8e14088cdf6b574d5e9ce6e30fc0'
```

Phew, that's a lengthy one. Hope this post helped you. Happy data sciencing!
