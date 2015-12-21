# apicius

_Recipes to build, sign and upload app container images_

Apicius is an early collection of recipes to build app container images (ACIs) compatible with the App Container specification, e.g. CoreOS's container runtime [Rocket](https://coreos.com/rkt). 

The first two available recipes are for the Nginx webserver and the PostgreSQL database. They are available from the [release page](https://github.com/namsral/apicius/releases).

I won't address the benifits of using ACIs except they're very lightweight, the nginx image is less than 2MB.


Getting started
---------------

To `run` these recipes you can either use the pre-built and signed images or clone the apicius repository and build your own. CoreOS has a great (getting started guide)[https://coreos.com/rkt/docs/latest/getting-started-guide.html) if you are not familiar with Rocket.

__Use pre-built images__

The images are signed and for rkt to veruify the signatures you must instruct rkt to trust the public key: 

    $ sudo rkt trust --prefix=namsral.com https://github.com/namsral/apicius/blob/gh-pages/dist/aci-pubkeys.gpg

Now you should be able to download and run the images:

    $ rkt fetch https://github.com/namsral/apicius/releases/download/beta/nginx-latest-linux-amd64.aci
    $ mkdir -p /data/pods/nginx/conf.d /data/pods/nginx/www
    $ sudo rkt run --port=http:80 --volume confd,kind=host,source=/data/pods/nginx/conf.d --volume www,kind=host,source=/data/pods/nginx/www,readOnly=false namsral.com/nginx


__Build your images__

In order to build your own images you can start with the recipes from the apicius repo:

    $ git clone https://github.com/namsral/apicius

Setup your environment file; you may also need to build the `acbuild` tool, see [Compile acbuild on CoreOS][]

    $ vi ./environment
    ...
    export PATH=/home/core/bin:$PATH
    ...

Make chances to the recipes if needed and run the build script:

    $ bash -c "source ./environment && ./build recipes/nginx"
    Bulding ACI in /tmp/tmp.61trBfduiy
    ...

The image will be built in `/tmp/tmp.61trBfduiy`.

    $ sudo rkt fetch /tmp/tmp.61trBfduiy/nginx-latest-linux-amd64.aci
    $ mkdir -p /data/pods/nginx/conf.d /data/pods/nginx/www
    $ sudo rkt run --port=http:80 --volume confd,kind=host,source=/data/pods/nginx/conf.d --volume www,kind=host,source=/data/pods/nginx/www,readOnly=false namsral.com/nginx


__Sign your images__

To sign your own images, you'll need to setup a GnuPG key, setup the GnuPG environment and use the `-s` flag:

    $ vi ./environment
    export GNUPGHOME=...
    ...
    $ bash -c "source ./environment && ./build -s recipes/nginx"
    $ sudo rkt trust --prefix=<your prefix> ./pubkeys.gpg


__Upload your images__

To upload your images to a GitHub repository release page you'll need to setup your GitHub environment, see the `environemnt` file.

    $ vi ./environment
    export GHTOKEN=...
    ...
    $ bash -c "source ./environment && ./build -u recipes/nginx"


Build Script
------------

__Requirements:__

- CoreOS _tested on CoreOS beta (877.1.0)_
- acbuild v0.2.1+git44c41e2 [githib.com](https://github.com/appc/acbuild)

CoreOS has all the depencies required to run the recipes including the dependancies to build `acbuild`.

__Options:__

    Usage: build [-ksuh] [recipe]
    Build, sign and upload ACI files.

    -k          keep the build directory
    -s          sign the built ACI file with gpg
    -u          upload the ACI and possible signature to a GitHub release
    -h          show help message


Compile acbuild on CoreOS
-------------------------

Run the following commands on a CoreOS host:

    $ cd $(mktemp -d)
    $ git clone https://github.com/appc/acbuild .
    $ ./build-docker

Include the new binary in your PATH:

    $ mkdir -p /home/core/bin
    $ cp ./bin/acbuild /home/core/bin/
    $ PATH=/home/core/bin:$PATH

_tested on CoreOS beta (877.1.0)_
