# Source-To-Image (S2I)

## Overview

[![Go Report Card](https://goreportcard.com/badge/github.com/openshift/source-to-image)](https://goreportcard.com/report/github.com/openshift/source-to-image)
[![GoDoc](https://godoc.org/github.com/openshift/source-to-image?status.png)](https://godoc.org/github.com/openshift/source-to-image)
[![Travis](https://travis-ci.org/openshift/source-to-image.svg?branch=master)](https://travis-ci.org/openshift/source-to-image)
[![License](https://img.shields.io/github/license/openshift/source-to-image.svg)](https://www.apache.org/licenses/LICENSE-2.0.html)

Source-to-Image (`S2I`) is a tool for building reproducible Docker images. `S2I` produces
ready-to-run images by injecting source code into a Docker image and *assembling*
a new Docker image for the application.  The result is then ready to use with `docker run`. 

The basic flow is:

1. Start with a ruby builder image which contains ruby, bundler, rake, apache, gcc, etc, all of which are needed to install and run ruby code.  
1. Launch a container using the builder image and injects the provided source code into a working directory within the container.
1. The container then transforms that source code into the appropriate runnable setup - in this case, by installing dependencies and moving the source code into a directory where Apache has been preconfigured to look for a Ruby application.
1. Commit the new container and set the image entrypoint to be a script (provided by the builder image) that will start Apache to host the Ruby application.

There are two primary patterns when building images:
* Produce an application image which includes all development/build tooling used to assemble the application as well as the runtime environment needed to execute the application.
* Produce an application image which only contains the final runnable application and the runtime environment.

The first pattern is appropriate for dynamic languages which typically need the same tools for building as execution.  The second can be used for compiled languages such as Java when it is undesirable to include build tools such as `javac` and `maven` in the runtime image.  Including build tools in the application image results in a self-contained image which can fully reproduce the application using the exact tools that originally created it.  More sophisticated flows make it possible to separate those two images into one image containing build inputs, build tools, and built artifacts and another image containing the built artifacts (provided as "source") and the runtime environment.

Try [building your own application image](#getting-started).

## Philosophy

1. Simplify the process of application source + builder image -> usable image for most use cases (the
   80%)
1. Define and implement a workflow for incremental builds that eventually uses only Docker
   primitives
1. Develop tooling that can assist in verifying that two different builder images result in the same
   `docker run` outcome for the same input
1. Use native Docker primitives to accomplish this - map out useful improvements to Docker that
   benefit all image builders
1. Allow users to build new application images in a controlled, secure, restricted environment

## Codified Image Construction Patterns

### Image flexibility   
S2I scripts can be written to inject application code into almost any existing Docker image, taking advantage of the existing image ecosystem.

### Speed   
With S2I, the assemble process can perform a large number of complex operations without creating a new layer at each step, resulting in a fast process. In addition, S2I scripts can be written to re-use artifacts stored in a previous version of the application image, rather than having to download or build them each time the build is run.

### Patchability  
S2I allows you to rebuild the application in a consistent way if an underlying image needs a patch due to a security issue.

### Operational security  
Building an arbitrary Dockerfile exposes the host system to root privilege escalation. This can be exploited by a malicious user because the entire Docker build process is run as a user with Docker privileges. S2I restricts the operations that can be performed as a root user and can run user
supplied build scripts as a non-root user.  Furthermore, operations teams can supply whitelisted builder images which will not perform
arbitrary actions such as installing random packages or downloading content from untrusted sources.

### Ecosystem
S2I encourages a shared ecosystem of images where you can leverage best practices for your applications and build processes.

### Reproducibility
Produced images can include all inputs used to produce the image, ensuring the image can be reproduced precisely including the versions
of build tools and dependencies.

## Anatomy of a builder image

Creating builder images is easy. `s2i` looks for you to supply the following scripts to use with an
image:

1. `assemble` - builds and/or deploys the source
1. `run`- runs the assembled artifacts
1. `save-artifacts` (optional) - captures the artifacts from a previous build into the next incremental build
1. `usage` (optional) - displays builder image usage information

Additionally for the best user experience and optimized `s2i` operation we suggest images
to have `/bin/sh` and `tar` commands available.

See a practical tutorial on how to create a builder image [here](examples/README.md) and read [this](https://github.com/openshift/source-to-image/blob/master/docs/builder_image.md) for a detailed description of the requirements and scripts along with examples of builder images.

## Build workflow

The `s2i build` workflow is:

1. `s2i` creates a container based on the build image and passes it a tar file that contains:
    1. The application source in `src`, excluding any files selected by `.s2iignore`
    1. The build artifacts in `artifacts` (if applicable - see [incremental builds](#incremental-builds))
1. `s2i` sets the environment variables from `.s2i/environment` (optional)
1. `s2i` starts the container and runs its `assemble` script
1. `s2i` waits for the container to finish
1. `s2i` commits the container, setting the CMD for the output image to be the `run` script and tagging the image with the name provided.

Filtering the contents of the source tree is possible if the user supplies a
`.s2iignore` file in the root directory of the source repository, where `.s2iignore` contains regular
expressions that capture the set of files and directories you want filtered from the image s2i produces.

Specifically:

1. Specify one rule per line, with each line terminating in `\n`.
1. Filepaths are appended to the absolute path of the  root of the source tree (either the local directory supplied, or the target destination of the clone of the remote source repository s2i creates).
1. Wildcards and globbing (file name expansion) leverage Go's `filepath.Match` and `filepath.Glob` functions.
1. Search is not recursive.  Subdirectory paths must be specified (though wildcards and regular expressions can be used in the subdirectory specifications).
1. If the first character is the `#` character, the line is treated as a comment.
1. If the first character is the `!`, the rule is an exception rule, and can undo candidates selected for filtering by prior rules (but only prior rules).

Here are some examples to help illustrate:

With specifying subdirectories, the `*/temp*` rule prevents the filtering of any files starting with `temp` that are in any subdirectory that is immediately (or one level) below the root directory.
And the `*/*/temp*` rule prevents the filtering of any files starting with `temp` that are in any subdirectory that is two levels below the root directory.

Next, to illustrate exception rules, first consider the following example snippet of a `.s2iignore` file:


```
*.md
!README.md
```


With this exception rule example, README.md will not be filtered, and remain in the image s2i produces.  However, with this snippet:


```
!README.md
*.md
```


`README.md`, if filtered by any prior rules, but then put back in by `!README.md`, would be filtered, and not part of the resulting image s2i produces.  Since `*.md` follows `!README.md`, `*.md` takes precedence.

Users can also set extra environment variables in the application source code.
They are passed to the build, and the `assemble` script consumes them. All
environment variables are also present in the output application image. These
variables are defined in the `.s2i/environment` file inside the application sources.
The format of this file is a simple key-value, for example:

```
FOO=bar
```

In this case, the value of `FOO` environment variable will be set to `bar`.

## Using ONBUILD images

In case you want to use one of the official Docker language stack images for
your build you don't have do anything extra. S2I is capable of recognizing the
Docker image with [ONBUILD](https://docs.docker.com/reference/builder/#onbuild) instructions and choosing the OnBuild strategy. This
strategy will trigger all ONBUILD instructions and execute the assemble script
(if it exists) as the last instruction.

Since the ONBUILD images usually don't provide any entrypoint, in order to use
this build strategy you will have to provide one. You can either include the 'run',
'start' or 'execute' script in your application source root folder or you can
specify a valid S2I script URL and the 'run' script will be fetched and set as
an entrypoint in that case.

### Incremental builds

`s2i` automatically detects:

* Whether a builder image is compatible with incremental building
* Whether a previous image exists, with the same name as the output name for this build

If a `save-artifacts` script exists, a prior image already exists, and the `--incremental=true` option is used, the workflow is as follows:

1. `s2i` creates a new Docker container from the prior build image
1. `s2i` runs `save-artifacts` in this container - this script is responsible for streaming out
   a tar of the artifacts to stdout
1. `s2i` builds the new output image:
    1. The artifacts from the previous build will be in the `artifacts` directory of the tar
       passed to the build
    1. The build image's `assemble` script is responsible for detecting and using the build
       artifacts

**NOTE**: The `save-artifacts` script is responsible for streaming out dependencies in a tar file.


## Dependencies

1. [Docker](http://www.docker.io) >= 1.6
1. [Go](http://golang.org/) >= 1.4
1. (optional) [Git](https://git-scm.com/)


## Installation

Assuming Go and Docker are installed and configured, execute the following commands:

```
$ go get github.com/openshift/source-to-image
$ cd ${GOPATH}/src/github.com/openshift/source-to-image
$ export PATH=$PATH:${GOPATH}/src/github.com/openshift/source-to-image/_output/local/bin/linux/amd64/
$ hack/build-go.sh
```

## Security

Since the `s2i` command uses the Docker client library, it has to run in the same
security context as the `docker` command. For some systems, it is enough to add
yourself into the 'docker' group to be able to work with Docker as 'non-root'.
In the latest versions of Fedora/RHEL, it is recommended to use the `sudo` command
as this way is more auditable and secure.

If you are using the `sudo docker` command already, then you will have to also use
`sudo s2i` to give S2I permission to work with Docker directly.

Be aware that being a member of the 'docker' group effectively grants root access,
as described [here](https://github.com/docker/docker/issues/9976).

## Getting Started

You can start using `s2i` right away (see [releases](https://github.com/openshift/source-to-image/releases))
with the following test sources and publicly available images:

```
$ s2i build git://github.com/pmorie/simple-ruby openshift/ruby-20-centos7 test-ruby-app
$ docker run --rm -i -p :8080 -t test-ruby-app
```

```
$ s2i build git://github.com/bparees/openshift-jee-sample openshift/wildfly-100-centos7 test-jee-app
$ docker run --rm -i -p :8080 -t test-jee-app
```

Interested in more advanced `s2i` usage? See [here](https://github.com/openshift/source-to-image/blob/master/docs/cli.md)
for detailed descriptions of the different CLI commands with examples.

Running into some issues and need some advice debugging?  Peruse [here](https://github.com/openshift/source-to-image/blob/master/docs/debugging-s2i.md) for some tips.
