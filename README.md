# Gradle Docker plugin

[![Build Status](https://drone.io/github.com/Transmode/gradle-docker/status.png)](https://drone.io/github.com/Transmode/gradle-docker/latest) [ ![Download](https://api.bintray.com/packages/transmode/gradle-plugins/gradle-docker/images/download.png) ](https://bintray.com/transmode/gradle-plugins/gradle-docker/_latestVersion)

This plugin for [Gradle](http://www.gradle.org/) adds the capability to build und publish [Docker](http://docker.io/) images from the build script.

## Extending the application plugin
The gradle-docker plugin adds a task `distDocker` if the project already has the [application plugin](http://www.gradle.org/docs/current/userguide/application_plugin.html) applied:

```gradle
apply plugin: 'application'
apply plugin: 'docker'
```

Executing the `distDocker` task builds a docker image containing all application files (libs, scripts, etc.) created by the `distTar` task from the application plugin. If you already use the application plugin to package your project then the docker plugin will add simple docker image building to your project.

By default `distDocker` uses a base image with a Java runtime according to the project's `targetCompatibility` property. The docker image entry point is set to the start script created by the application plugin. Checkout the [example](example/) project.

*Note: Only JVM based projects are supported.*


## Stand-alone
The docker plugin introduces the task type `Docker`. A task of this type can be used to build Docker images. See the [Dockerfile documentation](http://docs.docker.com/reference/builder/) for information about how docker containers are built.

The following example builds a docker image for the popular reverse proxy nginx. The image will be tagged with the name `foo/nginx`. The example is taken from the official Dockerfile [examples](http://docs.docker.com/reference/builder/#dockerfile-examples):

```gradle
apply plugin: 'docker'

buildscript {
    repositories { jcenter() }
    dependencies {
        classpath 'se.transmode.gradle:gradle-docker:1.1.1'
    }
}

group = "foo"

docker {
    baseImage "ubuntu"
    maintainer 'Guillaume J. Charmes "guillaume@dotcloud.com"'
}

task nginxDocker(type: Docker) {
    applicationName = "nginx"
    runCommand 'echo "deb http://archive.ubuntu.com/ubuntu precise main universe" > /etc/apt/sources.list'
    runCommand "apt-get update"
    runCommand "apt-get install -y inotify-tools nginx apache2 openssh-server"
}
```

## Configuring the plugin
The plugin exposes configuration options on 2 levels: globally through a plugin extension and on a per task basis. The plugin tries to always set sensible defaults for all properties. (The `maintainer` property is an exception. It is initialized with a useless default string.)

### Global configuration through plugin extension properties
Configuration properties in the plugin extension `docker` are applied to all Docker tasks. Available properties are:

 - `dockerBinary` - The path to the docker binary.
 - `baseImage` - The base docker image used when building images (i.e. the name after `FROM` in the Dockerfile).
 - `maintainer` - The name and email address of the image maintainer.
 - `registry` - The hostname and port of the Docker image registry unless the official Docker index is used.

Example to set the base docker image and maintainer name for all tasks:

```gradle
docker {
    maintainer = 'John Doe <john.doe@acme.org>'
    baseImage = 'johndoe/nextgenjdk:9.0'
}
```

### Task configuration through task properties
All properties that are exposed through the plugin extension can be overridden in each task.
The image tag is constructed according to:

```gradle
tag = "${project.group}/${applicationName}:${tagVersion}"
```

Where:

 - `project.group` -- This is a standard Gradle project property. If not defined, the `{project.group}/` is omitted.
 - `applicationName` -- The name of the application being "dockerized".
 - `tagVersion` -- Optional version name added to the image tag name.

The following example task will tag the docker image as `org.acme/bar:13.0`:

```gradle
...
group = 'org.acme'
...
task fooDocker(type: Docker) {
    applicationName = 'foobar'
    tagVersion = '13.0'
}
```

### A note about base images ###
If no base image is configured through the extension or task property a suitable image is chosen based on the project's `targetCompatibility`. A project targeting Java 7 will for instance get a default base image with a Java 7 runtime.

## Docker API vs. Command Line

By default the plug-in will use the `docker` command line tool to execute any docker commands (such as `build` and `push`).  However, it can be configured to use the Docker REST API instead via the `useApi` extension property:

    apply plugin: 'docker'

    docker {
        useApi true
    }

Use of the REST API requires that the Docker server be configured to listen over HTTP and that it have support for version 1.11 of the API (connecting over Unix Domain sockets is not supported yet).  The following configuration options are available:

* serverUrl -- set the URL used to contact the Docker server.  Defaults to `http://localhost:2375`
* username -- set the username used to authenticate the user with the Docker server.  Defaults to `nil` which means no authentication is performed.
* password -- set the password used to authenticate the user with the Docker server.
* email -- set the user's email used to authenticate the user with the Docker server.

For example:

    docker {
        useApi true
        serverUrl 'http://myserver:4243`
        username 'user'
        password 'password'
        email 'me@mycompany.com'
    }


## Requirements
* Gradle 2.x
* Docker 0.11+

You need to have docker installed in order to build docker images. However if the `dryRun` task property is set to `true`  all calls to docker are disabled. In that case only the Dockerfile and its context directory will be created.
#### Note to Gradle 1.x users
The plugin is built with Gradle 2.x and thus needs version 2.0 or higher to work due to a newer version of Groovy version included in Gradle 2.x (2.3 vs. 1.8.6). To use the plugin with Gradle 1.x you have to add Groovy's upward compatibility patch by adding the following line to your build file:

```gradle
buildscript {
    // ...
    dependencies {
         classpath 'se.transmode.gradle:gradle-docker:1.2'
         classpath 'org.codehaus.groovy:groovy-backports-compat23:2.3.5'
    }
}
```

