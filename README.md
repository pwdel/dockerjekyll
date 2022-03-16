# dockerjekyll

Running Jekyll on Docker

## Background


## The Jekyll Docker Image

There is a Jeklyll Docker image, the versions of which can be found [here](https://hub.docker.com/r/jekyll/jekyll/tags).

Typically in this scenario we would go about creating a Dockerfile, and then a docker-compose.yml to customize our setup in a delcarative way. We notice that there is a specific version and sub-version number of jekyll available at, ```docker pull jekyll/jekyll:4.2.0``` to keep things as stable as possible.

However, once we have everything organized properly, and attempt to build the image with the following tag:

```
docker build -t dockerjekyll_image:latest .
```

We get an error, specifically:

```
------
 > resolve image config for docker.io/library/dockerfile:1:
------
failed to solve with frontend dockerfile.v0: failed to solve with frontend gateway.v0: pull access denied, repository does not exist or may require authorization: server message: insufficient_scope: authorization failed
```

However, the following works in terms of instantly turning all of the containing files into a site per the, `jekyll s` command:

```
export JEKYLL_VERSION=4
docker run --rm \
  --volume="$PWD:/srv/jekyll" \
  -it jekyll/jekyll:$JEKYLL_VERSION \
  jekyll build
```

It's not very clear why this error is happening.  The first thing that is suggested on Stackoverflow is to go to, ```~/.docker``` and then to list all files, "ls -a" and to remove the .token_seed and .token_seed.lock.  But is this really necessary?

### What is the Docker token_seed?

First off, what is the Docker token_seed?

Docker Hub lets you create personal access tokens as alternatives to your password. You can use tokens to access Hub images from the Docker CLI.

Using personal access tokens provides some advantages over a password:

* You can investigate the last usage of the access token and disable or delete it if you find any suspicious activity.
* When using an access token, you can’t perform any admin activity on the account, including changing the password. It protects your account if your computer is compromised.

That being said it appears we may not have been using a Docker Hub token, because we never set one up in the first place, so deleting it may not have done anything either way.

### Paradigm Shift

Typically when we're using Docker, the idea is that a Dockerfile is used to build an image locally on a machine, and then a docker-compose is used to build that image in a way that we want.

In this scenario, since the Dockerfile doesn't work, but the, ```docker run``` command does, and due to the nature of how Jekyll normally works, its' reasonable to understand that the Docker Hub settings configured by Jekyll are set such that only the, "docker run," command is workable, e.g. it's designed to run within an environment in parallel to written code, as a way to compile the code into static website code, thereby setting up a bind mount and a port for us, rather than having us design the Dockerfile and docker-compose setup ourselves.

There are two main offerings: Jekyll Server and Jekyll Builder.  Note that in both instances, we are asked to use the, "docker run" command and mount the actual server as a volume, e.g. ```--volume="$PWD:/srv/jekyll" \``` rather than mount our code as a volume.
#### Jekyll Builder

Jekyll Builder is designed to be used by those who are deploying Jekyll builds to a seperate server via a Continous Integration. It includes openssh, lftp, and other packages.  The usage is:

```
export JEKYLL_VERSION=3.8
docker run --rm \
  --volume="$PWD:/srv/jekyll" \
  -it jekyll/builder:$JEKYLL_VERSION \
  jekyll build
```

#### Jekyll Server

Jekyll Server is analogous to running, "jekyll s" and allows jekyll to run in server mode inside of a container to rebuild a site and provides access through a web server. The default port is localhost:4000 in a web server.

```
docker run --rm \
  --volume="$PWD:/srv/jekyll" \
  --publish [::1]:4000:4000 \
  jekyll/jekyll \
  jekyll serve
```
## Pulling Everything Together and Creating Our Own Script

So in a typical scenario if we wanted to create mountable code, we would create a folder called, "app," hierarchically below our Dockerfile, while using a COPY command in the Dockerfile to strap that code into our container, and within our docker-compose.yml we would bind-mount said code into a given position on the container, like so:

```
    volumes:
      - type: bind
        source: ./app
        target: /home/app

```

However in this scenario, the script is flipped and the Jekyll server is being mounted as a volume, leaving our code alone on our local machine, with the Jekll server temporarily opening up and building the code, where it can be viewed as a static site.

This makes sense because after all - Jekyll is a static site builder, not an application builder.

At any rate, if we want to give our selves some more static control over the command, e.g. make it so that we are less likely to accidentally type the command wongly into the command prompt, we can always create our own script to run whatever set of jekyll commands we would like.

So we set up the following script, which basicaly runs the jekyll server on the localhost:4000 port:

```
#!/usr/bin/env bash

# set exit if fails, any subsequent commands will fail and shell exits immediately
set -e

# cd into website before running jekyll
cd website

echo "Setting up jekyll website..."

# set the Jekyll version
export JEKYLL_VERSION=4.0
# run jekyll server version, e.g. /srv/jekyll mounted as a volume
docker run -d -p 4000:4000 --rm           \
  --name dockerjekyll_container           \
  --volume="$PWD:/srv/jekyll"             \
  -it jekyll/builder:$JEKYLL_VERSION      \
  jekyll serve
```

When we run the above script, we can see a fairly verbose output which in fact tells us where in the process of running the Jekyll server we are at and what the server address is:

```
$ ./runjekyll
Setting up jekyll website...
Fetching gem metadata from https://rubygems.org/
Fetching gem metadata from https://rubygems.org/.........
Fetching gem metadata from https://rubygems.org/.........
Using public_suffix 4.0.6
Using bundler 2.2.24
Using colorator 1.1.0
Using concurrent-ruby 1.1.9
Using eventmachine 1.2.7
Fetching http_parser.rb 0.8.0
Fetching ffi 1.15.5
Installing http_parser.rb 0.8.0 with native extensions
Installing ffi 1.15.5 with native extensions
Using forwardable-extended 2.6.0
Fetching rb-fsevent 0.11.1
Installing rb-fsevent 0.11.1
Fetching rexml 3.2.5
Installing rexml 3.2.5
Using liquid 4.0.3
Using mercenary 0.4.0
Fetching rouge 3.28.0
Installing rouge 3.28.0
Using safe_yaml 1.0.5
Fetching unicode-display_width 1.8.0
Installing unicode-display_width 1.8.0
Fetching webrick 1.7.0
Installing webrick 1.7.0
Using addressable 2.8.0
Fetching i18n 1.10.0
Installing i18n 1.10.0
Fetching em-websocket 0.5.3
Installing em-websocket 0.5.3
Using pathutil 0.16.2
Using kramdown 2.3.1
Using terminal-table 2.0.0
Using kramdown-parser-gfm 1.1.0
Using sassc 2.4.0
Using rb-inotify 0.10.1
Fetching jekyll-sass-converter 2.2.0
Fetching listen 3.7.1
Installing listen 3.7.1
Installing jekyll-sass-converter 2.2.0
Using jekyll-watch 2.2.1
Fetching jekyll 4.2.2
Installing jekyll 4.2.2
Fetching jekyll-feed 0.16.0
Fetching jekyll-seo-tag 2.8.0
Installing jekyll-feed 0.16.0
Installing jekyll-seo-tag 2.8.0
Using jekyll-sitemap 1.4.0
Bundle complete! 4 Gemfile dependencies, 32 gems now installed.
Use `bundle info [gemname]` to see where a bundled gem is installed.
ruby 2.7.1p83 (2020-03-31 revision a0c7c23c9c) [x86_64-linux-musl]
Configuration file: /srv/jekyll/_config.yml
            Source: /srv/jekyll
       Destination: /srv/jekyll/_site
 Incremental build: disabled. Enable with --incremental
      Generating...
       Jekyll Feed: Generating feed for posts
                    done in 1.081 seconds.
 Auto-regeneration: enabled for '/srv/jekyll'
    Server address: http://0.0.0.0:4000/
  Server running... press ctrl-c to stop.
```