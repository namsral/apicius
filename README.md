# Apicius

#### A tool to build, sign and upload app container images

Apicius is a build tool with an early collection of predefined recipes to build app container images (ACIs) compatible with the App Container specification, e.g. CoreOS's container runtime [Rocket]. I won't address the benifits of using ACIs except they're very lightweight which makes them fast to build, load and boot. As an example, the pre-built Nginx container is 1.3MB, the equivalent PostgreSQL container is 6.2MB. These pre-built app container images are available on the [apicius release page][apicius-releases].

The idea is to clone this repository, create some recipes which hook-up to your CI service and start pushing app container images to your container scheduler. The `apicius` tool is just  a convenience tool to automate the building, signing and uploading of your containers.

#### Requirements:

Apicius and its recipes are tested to run on [CoreOS]. Other Linux systems which support `systemd-nspawn` should work but your milage may vary.

[apicius-releases]: https://github.com/namsral/apicius/releases
[CoreOS]: https://coreos.com
[Rocket]: https://coreos.com/rkt

#### Full example:

    $ apicius --sign --upload github recipes/nginx

#### Usage:

    Usage: apicius [-ehksu] [-u service] [recipe]
    Build, sign and upload app container images (ACI)                             
                                                                                  
            -e              load environment from file
            -h              show help message                                     
            -k              keep the temporary build directory                    
            -s              generate a signature with GnuGP                       
            -u service      upload image and signature, available services: github


Recipes
-------

#### What is a recipe?

A recipe is a directory containing the main build executable and any content needed by the build script. The contents of the `recipes/nginx` recipe:

    ./build
    ./extra/nginx.conf

The build executable for the `recipes/nginx` recipe is a bash script:

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
    ...

This script runs the [acbuild] tool from CoreOS to build the app container image.

[acbuild]: https://github.com/appc/acbuild
[acbuild-releases]: https://github.com/appc/acbuild/releases


Getting started
---------------

You can either run the pre-built images from the [release page][apicius-releases] or build your own.

The following examples assume you're using Rocket on CoreOS. Guides for both are available on [coreos.com][CoreOS].

#### Use pre-built images

The pre-built image are signed, and for Rocket to validate the signatures trust the public key in order to verify the image's signature: 

    $ sudo rkt trust --prefix=namsral.com https://github.com/namsral/apicius/blob/gh-pages/dist/aci-pubkeys.gpg

Fetch and run the images:

    $ rkt fetch https://github.com/namsral/apicius/releases/download/beta/nginx-latest-linux-amd64.aci
    $ mkdir -p /data/pods/nginx/conf.d /data/pods/nginx/www
    $ sudo rkt run --port=http:80 --volume confd,kind=host,source=/data/pods/nginx/conf.d --volume www,kind=host,source=/data/pods/nginx/www,readOnly=false namsral.com/nginx

#### Build your own images

Install the `acbuild` tool from the [acbuild release page][acbuild-releases] or [compile it yourself](#compile-acbuild-on-coreos) using Docker.

In order to build your own images you can start with the recipes from the `apicius` repo:

    $ git clone https://github.com/namsral/apicius .

Setup your environment file; 

    $ vi ./environment
    ...
    export PATH=/home/core/bin:$PATH
    ...

Make chances to the recipes if needed and run the build script:

    $ ./apicius recipes/nginx
    Bulding ACI in /tmp/tmp.61trBfduiy
    ...
    $ ls -1
    nginx-latest-linux-amd64.aci


#### Sign your images

To sign your images, you'll need to setup a GnuPG key, import the public key in Rocket and setup the GnuPG environment:

    $ sudo rkt trust --prefix=<your prefix> ./pubkeys.gpg
    $ vi ./environment
    ...
    export GNUPGHOME=...
    ...
    $ apicious -e ./environment -s recipes/nginx
    $ ls -1
    nginx-latest-linux-amd64.aci
    nginx-latest-linux-amd64.aci.asc

Now you're ready to load and run your app container image.

    $ rkt fetch ./nginx-latest-linux-amd64.aci
    $ mkdir -p /data/pods/nginx/conf.d /data/pods/nginx/www
    $ sudo rkt run --port=http:80 --volume confd,kind=host,source=/data/pods/nginx/conf.d --volume www,kind=host,source=/data/pods/nginx/www,readOnly=false namsral.com/nginx

Or upload them to your storage service for your container runtime to fetch them.

#### Upload your images and signatures to a GutHub release page

To upload your images to a GitHub release page you'll need to setup your GitHub environment, see the `environment` file.

    $ vi ./environment
    ...
    export GHTOKEN=...
    ...
    $ apicious -e ./environment -s -u github recipes/nginx
    ...


Other
-------

#### Compile acbuild on CoreOS

Run the following commands on a CoreOS host:

    $ cd $(mktemp -d)
    $ git clone https://github.com/appc/acbuild .
    $ ./build-docker

Include the new binary in your PATH:

    $ mkdir -p /home/core/bin
    $ cp ./bin/acbuild /home/core/bin/
    $ PATH=/home/core/bin:$PATH

_tested on CoreOS beta (877.1.0)_
