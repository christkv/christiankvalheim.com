---
author: "Christian Kvalheim"
date: 2014-09-25
title: Diagnosing driver installation problems
weight: 10
authorAvatar: hugo-logo.png
keywords: ["Development","MongoDB","Node.js","github","mongodb","js","Javascript"]
tags: ["mongodb", "drivers", "node.js"]
slug: "an_introduction_to_1_4_and_2_6"
section: "post"
---

# The MongoDB driver is not installing correctly
Most of the problems are with the native BSON parser and there can be a multitude of reasons why it's not installing correctly and the driver is falling back to the js implementation of the serializer. This post will hopefully help you track down what the problem is on your system. We will start looking at it from a checklist point of view.

## Build Essentials Linux/Unix
If you don't have the build essentials it won't build. In the case of linux you will need gcc and g++, node.js with all the headers and python. The easiest way to figure out what's missing is by trying to build the js-bson project. You can do this by performing the following steps.

    git clone https://github.com/mongodb/js-bson.git
    cd js-bson
    npm install
    make test

If all the steps complete you have the right toolchain installed. If you get node-gyp not found you need to install it globally by doing.

    npm install -g node-gyp

If correctly compile and runs the tests you are golden. We can now try to install the mongod driver by performing the following command.

    cd yourproject
    npm install mongod

If it still fails the next step is to examine the npm log. Rerun the command but in this case in verbose mode.

    npm --loglevel verbose install mongodb

This will print out all the steps npm is performing while trying to install the module.

## Build Essentials Windows
In windows there are pre-compiled binaries for bson. **However** these only work on 0.10.x or earlier as 0.11.x contains the completely breaking v8 API. So if you want to run 0.11.x you will have to set up your own build toolchain. So far I've only had luck with the following combination.

    Visual Studio c++ 2010 (do not use higher versions)
    Windows 7 64bit SDK
    Python 2.7 or higher

Unfortunately you cannot easily use the make file under windows (I have not been able to do so correctly using MinGw).

Open visual studio command prompt. Ensure node.exe is in your path and install node-gyp.

    npm install -g node-gyp

Next you will have to build the project manually to test it. Use any tool you use with git and grab the repo.

    git clone https://github.com/mongodb/js-bson.git
    cd js-bson
    npm install
    node-gyp rebuild

This should rebuild the driver successfully if you have everything set up correctly.

## Other possible issues
Your python installation might be hosed making gyp break. I always recommend that you test your deployment environment first by trying to build node itself on the server in question as this should unearth any issues with broken packages (and there are a lot of broken packages out there).

Another thing is to ensure your user has write permission to wherever the node modules are being installed.