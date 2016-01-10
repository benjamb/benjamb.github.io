---
published: true
title: Building the Genivi Demo Platform with Concourse.ci
layout: post
---
So, just before the Christmas period I began trying to build the Yocto GENIVI Demo Platform using Concourse.ci; sounds simple enough, just get something running that has git and can run bitbake and everything will just work.

To get started with Concourse, take a look at the [Getting Started](http://concourse.ci/getting-started.html) page.

tl;dr:

  * `vagrant init concourse/lite`
  * `vagrant up`
  * observe shiny web page at 192.168.100.4:8080
  * grab the fly cli from the web page, link at the bottom-right corner

After some reading I created a simple pipeline with the GDP repo as a resource (save this YAML as pipeline.yml):

<pre>
resources:
- name: genivi-demo
  type: git
  source:
    uri: http://git.projects.genivi.org/genivi-demo-platform.git
    branch: qemux86-64

jobs:
- name: yocto-demo
  plan:
  - get: genivi-demo
    params: {submodules: all}
    trigger: true
  - task: build
    config:
      platform: linux
      image: "docker:///ubuntu"
      inputs:
        - name: genivi-demo
      run:
        path: bash
	args:
          - -c
          - |
            set -e
            pushd genivi-demo
              source init.sh
              touch conf/sanity.conf
              bitbake genivi-demo-platform
            popd
</pre>

After running `fly set-pipeline -p demo-pipeline -c pipeline.yml`, the web page will now display this very basic pipeline.

However, the above won't run successfully as bitbake isn't available on the image being used.  To fix this I created my own docker image using the following Dockerfile:

<pre>
FROM ubuntu:14.04

RUN apt-get update && \
    apt-get install -y gawk wget git-core diffstat unzip texinfo \
    gcc-multilib build-essential chrpath socat libsdl1.2-dev xterm \
    python-ply python-progressbar python-bs4 python-pyinotify

# Install bitbake
ENV HOME /root
WORKDIR $HOME
RUN git clone git://git.openembedded.org/bitbake.git --branch 1.26.0 bitbake
WORKDIR bitbake
RUN python ./setup.py install
WORKDIR $HOME
RUN rm -rf bitbake
</pre>

The result is benbrown/yocto-demo on Docker Hub, to create the image yourself:

* docker build -t user/yocto-demo .

from the directory you saved the Dockerfile.

Amend the previously given YAML with the following:

<pre>
-      image: "docker:///ubuntu"
+      image: "docker:///benbrown/yocto-demo"
</pre>

We can now run the build task once the pipeline has been updated, this should result in a successful build of the GDP, however it doesn't publish artefacts, nor does it save DL_DIR or SSTATE_DIR so rebuilds will be done from scratch.