
# This article got updated

# Cloud native continuous delivery using Paketo buildpacks

Doing a "git push" and be done with deployment has been a developer utopia ever since Heroku popularized it.  The good news is, we can do all of this within our Kubernetes cluster. Buildpacks provide a rich ecosystem where you can build your application's container images without messing up with writing Dockerfiles and worrying about security.

In this article, I'll walk you through how to build and deploy a Drupal 8 application from the git push stage to shipping to production.


## Why buildpacks?


## First cut

Prerequisites:

1.  Drupal 8 project/codebase. Preferably this, or your own.
2.  Pack cli. Instructions to download [here](https://paketo.io/docs/)
3.  Docker

Let's get the source code first.

    composer create-project "drupal/recommended-project:^8" d8-buildpack

Pack needs answers to 2 questions to build a container image.

What are you going to build? - The answer's mostly your application source code.

How are you going to build it?
This is not a one-size-fits-all. Case in point, our Drupal 8 is a PHP application will be built by running `composer install` to manage the dependencies. A Drupal 7 has no notion of composer, but it's still a PHP application.For some PHP applications, we might use Nginx, for others Apache etc.

The buildpack specification provides a set of images which can be assembled in a required sequence. This allows us to tailor our build process down to the tiniest detail.

Now, think of each of these buildpacks as a part of a build "pipeline", where 4 processes are run, in that order:

1.  Detection - check if this buildpack is relevant to building the app
2.  Analysis - see how this buildpack run can be optimzed
3.  Build - the actual build process, in our case, running composer install
4.  Export - The final OCI image of this buildpack.

This is termed "Lifecycle" in the Buildpack spec document.

Finally, we have a "stack", which specifies what image to use for building the app, and what image to use for running the app.

A builder is a combination of all the available buildpacks, the lifecycle and a stack.

Armed with this concept, let's take a stab at how we will build our code.

First, we pick and choose what builder image we want to use.

    $ pack builder suggest
    Suggested builders:
            Google:                gcr.io/buildpacks/builder:v1      Ubuntu 18 base image with buildpacks for .NET, Go, Java, Node.js, and Python                                              
            Heroku:                heroku/buildpacks:18              heroku-18 base image with buildpacks for Ruby, Java, Node.js, Python, Golang, & PHP                                       
            Paketo Buildpacks:     paketobuildpacks/builder:base     Ubuntu bionic base image with buildpacks for Java, .NET Core, NodeJS, Go, Ruby, NGINX and Procfile                        
            Paketo Buildpacks:     paketobuildpacks/builder:full     Ubuntu bionic base image with buildpacks for Java, .NET Core, NodeJS, Go, PHP, Ruby, Apache HTTPD, NGINX and Procfile     
            Paketo Buildpacks:     paketobuildpacks/builder:tiny     Tiny base image (bionic build image, distroless-like run image) with buildpacks for Java Native Image and Go 

The `paketobuildpacks/builder:full` builder looks like a good bet, as it contains buildpacks for PHP and Nginx.

We can build an image using `pack build`.

    $ pack build --builder paketobuildpacks/builder:full drupal-8

Let's dissect the output of the above command.

    ===> DETECTING
    3 of 5 buildpacks participating
    paketo-buildpacks/php-dist     0.1.0
    paketo-buildpacks/php-composer 0.1.1
    paketo-buildpacks/php-web      0.0.136

    ===> ANALYZING
    Previous image with name "drupal-8" not found
    ===> RESTORING
    ===> BUILDING
    Paketo PHP Buildpack 0.1.0
      Resolving PHP version
        Candidate version sources (in priority order):
          default-versions -> "7.4.*"
                           -> ""
    
        Selected PHP version (using default-versions): 7.4.13
    
      Executing build process
        Installing PHP 7.4.13
          Completed in 8.618s

    Paketo PHP Composer Buildpack 0.1.1
       2.0.7: Contributing to layer
        Downloading from https://buildpacks.cloudfoundry.org/dependencies/composer/composer_2.0.7_linux_noarch_any-stack_0a060e8c.phar
        Verifying checksum
        Expanding to /layers/paketo-buildpacks_php-composer/composer
      PHP Composer Cache e57a3520f2fcb109f78c013afc32e8c9178ebda4c9b9e14a706c666172c8902a: Contributing to layer
    No vendor dir present, checking platform requirements from the lock file
      PHP Composer 383e93da1dc777228cc8c79de84b7468d32205fbd9f43997dadfdfc14850e812: Contributing to layer
    Installing dependencies from lock file
    Verifying lock file contents can be installed on current platform.
    Package operations: 59 installs, 0 updates, 0 removals
      - Downloading composer/installers (v1.10.0)
      - Downloading drupal/core-composer-scaffold (8.9.13)
      - Downloading drupal/core-project-message (8.9.13)
    ....

    Paketo PHP Web Buildpack 0.0.136
      PHP Web 30e9170f9f2e0247b478cacacff15e9b26ee4377b2aeb5840778a8d86b303e8b: Contributing to layer
      Configuring PHP Application
        Using feature -- PHP
        Writing PHPRC to shared
        Writing PHP_INI_SCAN_DIR to shared
        Using feature -- Scripts
        Buildpack could not find a file to execute. Either set php.script in buildpack.yml or include one of these files [app.php, main.php, run.php, start.php]
      Process types:
        task: php /workspace/app.php
        web:  php /workspace/app.php

    ===> EXPORTING
    Reusing layer 'paketo-buildpacks/php-dist:php'
    Adding layer 'paketo-buildpacks/php-composer:php-composer-packages'
    Reusing layer 'paketo-buildpacks/php-web:php-web'
    ...
    Successfully built image drupal-8

Let's take the newly built image for a spin.

    $ docker run drupal-8
    Could not open input file: /workspace/app.php

&#x2026; and rightly so. The most common way to run a PHP application is to front is via a webserver like Apache or Nginx. In our case, let's go with Nginx.

The "buildpack-way" to specify that we're using Nginx instead of running a PHP process, is to create a `buildpack.yml` file at the top level directory, as documented [here](https://github.com/paketo-buildpacks/php-web#buildpackyml-configurations).

    ---
    php:
      version: 7.4.*
      webserver: nginx
      webdirectory: web

The build process reads and parses this file, and infers the following:

1.  The application will be using the latest stable version of PHP 7.4
2.  The application will use Nginx to serve the PHP code.
3.  The application directory will be `web`.

Building the image again, we notice 2 changes.

    ===> DETECTING
    4 of 5 buildpacks participating
    paketo-buildpacks/nginx        0.0.194
    paketo-buildpacks/php-dist     0.1.0
    paketo-buildpacks/php-composer 0.1.1
    paketo-buildpacks/php-web      0.0.136

It detects and adds an additional Nginx buildpack.

    ===> BUILDING
    Paketo Nginx Server Buildpack 0.0.194
      Resolving Nginx Server version
        Candidate version sources (in priority order):
          <unknown> -> "*"
    
        Selected Nginx Server version (using <unknown>): 1.19.4
    
      Executing build process
        Installing Nginx Server 1.19.4
          Completed in 2.57s

It installs Nginx as part of the final image.

    Paketo PHP Web Buildpack 0.0.136
      PHP Web 232c761885e0c69e36fa273694b1b2f20ff34b0a05a4fb4826c2077a29e409f8: Contributing to layer
      Configuring PHP Application
        Using feature -- PHP
        Writing PHPRC to shared
        Writing PHP_INI_SCAN_DIR to shared
        Using feature -- Nginx
        Using feature -- PhpFpm
        Using feature -- ProcMgr
      Process types:
        web: procmgr /layers/paketo-buildpacks_php-web/php-web/procs.yml

The build lifecycle of PHP Web figures out that the right way to run this image is to use Nginx + PHP FPM and create a process.

The rest of the steps are essentially unchanged.

Let's run the revised image.

    $ docker run --interactive --tty --env PORT=8080  --publish 8080:8080 drupal-8

When we hit localhost:8080 on the browser, we're greeted with a 500 error.

The logs indicate that PHP is not able to find the vendor directory dependencies.

    2021/02/02 09:45:17 [error] 25#0: *1 FastCGI sent in stderr: "PHP message: PHP Warning:  include(/workspace/web/core/lib/Drupal/Core/DrupalKernel.php): failed to open stream: No such file or directory in /layers/paketo-buildpacks_php-composer/php-composer-packages/vendor/composer/ClassLoader.php on line 444PHP message: PHP Warning:  include(): Failed opening '/workspace/web/core/lib/Drupal/Core/DrupalKernel.php' for inclusion (include_path='/layers/paketo-buildpacks_php-composer/php-composer-packages/vendor/pear/pear_exception:/layers/paketo-buildpacks_php-composer/php-composer-packages/vendor/pear/console_getopt:/layers/paketo-buildpacks_php-composer/php-composer-packages/vendor/pear/pear-core-minimal/src:/layers/paketo-buildpacks_php-composer/php-composer-packages/vendor/pear/archive_tar:/layers/paketo-buildpacks_php-dist/php/lib/php:/workspace/lib') in /layers/paketo-buildpacks_php-composer/php-composer-packages/vendor/composer/ClassLoader.php on line 444PHP message: PHP Fatal error:  Uncaught Error: Class 'Drupal\Core\DrupalKernel' not found in /workspace/web/index.php:16

Let's dive into the container real quick and see how it's built.


## Some stack specific quirks

Restructuring buildpacks.

## Creating a custom build process


## Testing our image


## Repeat everything in the cluster

## Tekton vs Kpack

## Why build container images in Kubernetes?

## How to update builder images

By tagging them and running a cron task to fetch latest versions.

## Conclusion

kp cli

Helm release

Add a helm wait.

Tekton trigger.

Get build logs.
