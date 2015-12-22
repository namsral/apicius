# Apicius

#### A tool to build, sign and upload app container images

Apicius is a build tool with an early collection of predefined recipes to build app container images (ACIs) compatible with the App Container specification, e.g. CoreOS's container runtime [Rocket](https://coreos.com/rkt). I won't address the benifits of using ACIs except they're very lightweight which makes them fast to build, load and run. As an example, the pre-built Nginx container is 1.3MB, the equivalent PostgreSQL container is 6.2MB. These pre-built app container images are available on the [release page](https://github.com/namsral/apicius/releases).

The idea is to clone this repository, create some recipes which hook-up to your CI service and start pushing app container images to your container scheduler. The `apicius` tool is just  a convenience tool to automate the building, signing and uploading of your containers.

Apicius and its recipes are tested on [CoreOS] and is the easiest way to get started. Other Linux systems which support systemd-nspawn should work but your milage may vary.

[acbuild]: (https://github.com/appc/acbuild/releases "a build tool for ACIs")
[CoreOS]: (coreos.com)

__Full example:__

    $ bash -c "source ./environment && ./build --sign --upload github recipes/nginx"

__Usage:__

    Usage: apicius [-hks] [-u service] [recipe]                                   
    Build, sign and upload app container images (ACI)                             
                                                                                  
            -h              show help message                                     
            -k              keep the temporary build directory                    
            -s              generate a signature with GnuGP                       
            -u service      upload image and signature, available services: github


Recipes
-----------

__What is a recipe?__

The contents of a `apicius` recipe is a directory holding extra content for the container and a build script:

    ./build
    ./extra/nginx.conf

The build script for Nginx is written in bash and uses the `acbuild` tool:

    #!/usr/bin/env bash
    set -e

    # Start the build with an empty ACI
    acbuild --debug begin
    ...
    # Name the ACI
    acbuild --debug set-name namsral.com/nginx
    ...
    # Install nginx
    acbuild --debug run apk update
    acbuild --debug run apk add nginx
    ...
    # Copy files
    acbuild --debug copy ./extra/nginx.conf /etc/nginx/nginx.conf


Getting started
---------------

To run these recipes you can either use the pre-built and signed images or clone the apicius repository and build your own. CoreOS has a great [getting started guide](https://coreos.com/rkt/docs/latest/getting-started-guide.html) if you are not familiar with Rocket.

__Use pre-built images__

Trust the public key in order to verify the image's signature: 

    $ sudo rkt trust --prefix=namsral.com https://github.com/namsral/apicius/blob/gh-pages/dist/aci-pubkeys.gpg

Fetch and run the images:

    $ rkt fetch https://github.com/namsral/apicius/releases/download/beta/nginx-latest-linux-amd64.aci
    $ mkdir -p /data/pods/nginx/conf.d /data/pods/nginx/www
    $ sudo rkt run --port=http:80 --volume confd,kind=host,source=/data/pods/nginx/conf.d --volume www,kind=host,source=/data/pods/nginx/www,readOnly=false namsral.com/nginx


__Build your own images__

Install the `acbuild` tool from the [acbuild release page][acbuild] or [compile it yourself](#compile-acbuild-on-coreos) using Docker.

In order to build your own images you can start with the recipes from the `apicius` repo:

    $ git clone https://github.com/namsral/apicius .

Setup your environment file; 

    $ vi ./environment
    ...
    export PATH=/home/core/bin:$PATH
    ...

Make chances to the recipes if needed and run the build script:

    $ bash -c "source ./environment && ./build recipes/nginx"
    Bulding ACI in /tmp/tmp.61trBfduiy
    ...
    $ ls -1
    nginx-latest-linux-amd64.aci

The image will be built in `/tmp/tmp.61trBfduiy`.

__Sign your images__

To sign your own images, you'll need to setup a GnuPG key, trust your public key and setup the GnuPG environment.

    $ sudo rkt trust --prefix=<your prefix> ./pubkeys.gpg
    $ vi ./environment
    ...
    export GNUPGHOME=...
    ...
    $ bash -c "source ./environment && ./build -s recipes/nginx"
    $ ls -1
    nginx-latest-linux-amd64.aci
    nginx-latest-linux-amd64.aci.asc

__Upload your images and signatures__

To upload your images to a GitHub repository release page you'll need to setup your GitHub environment, see the `environemnt` file.

    $ vi ./environment
    ...
    export GHTOKEN=...
    ...
    $ bash -c "source ./environment && ./build -s -u github recipes/nginx"


Other
-------

__Compile `acbuild` on CoreOS__

Run the following commands on a CoreOS host:

    $ cd $(mktemp -d)
    $ git clone https://github.com/appc/acbuild .
    $ ./build-docker

Include the new binary in your PATH:

    $ mkdir -p /home/core/bin
    $ cp ./bin/acbuild /home/core/bin/
    $ PATH=/home/core/bin:$PATH

_tested on CoreOS beta (877.1.0)_

