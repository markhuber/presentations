#modern-ie-vagrant: A New Way To Test IE Browsers

Microsoft, as part of their Modern IE project for developers (now known as Microsoft Edge, their new browser), has released a series of virtual machine images covering everything from IE6 & Windows XP (shudder) all the way up to IE 11 & Windows 10. They've released these in a variety of formats from their own virtualization technology HyperV all the way to Oracle Virtualbox. All of these can be downloaded from http://dev.modern.ie/tools/vms/

Not satisfied with the workflow of downloading these images (all 35GB worth), I set out to leverage Vagrant to make it even easier to get started with these virtual machines. The result is [modern-ie-vagrant](https://github.com/markhuber/modern-ie-vagrant).

##TL;DR

To get started, you'll need [Oracle Virtualbox](http://virtualbox.org) and [Vagrant](https://www.vagrantup.com/) installed. 

From there all you have to do is clone our git repository [modern-ie-vagrant](https://github.com/markhuber/modern-ie-vagrant) or you can download it as a zip file [here](https://github.com/markhuber/modern-ie-vagrant/archive/master.zip). 

```bash
git clone https://github.com/markhuber/modern-ie-vagrant.git
```

If you've never used Vagrant before, the process is pretty straight forward. Using your favorite command-line application, work your way into the directory you cloned from GitHub and use any of the following commands.

To list the available virtual machines:

```bash
vagrant status
```

To start a new virtual machine

```bash
vagrant up IE10-Win7
```

To start multiple virtual machines at once

```bash
vagrant up IE11-Win7 IE10-Win7 IE9-Win7 IE8-Win7
```

To power down a virtual machine from the command line

```bash
All machines: vagrant halt
Specific machine: vagrant halt IE10-Win7
```

To reclaim some disk space, you can delete all or some of the images with

```bash
All machines: vagrant destroy
Specific machine: vagrant destroy IE10-Win7
```

You can purge the cache of downloaded machine images with

```bash
vagrant box list
vagrant box remove [boxname]
```

Vagrant handles retrieving the machine images on demand. The first time you use an image, it will automatically download it from the internet which may take around 20 minutes. Once you've retrieved the image once, it is cached and creating machines takes under 2 minutes, while restarting an already created image can be as quick as 30 seconds.

##Too much typing? Try Vagrant Manager GUI

I've been testing this project with [Vagrant Manager](http://vagrantmanager.com/) with great success. The first time you use Vagrant Manager, if you have not already created a virtual machine for testing from the command-line, it will not automatically find your project. You can use their bookmark feature to tell it where to find the Vagrantfile. From here, you can create and destroy machines right from your toolbar. This works on both Mac OS X and Windows.

##Creating modern-ie-vagrant and docker-vagrant-catalog

Creating this project was an unexpected wealth of learning opportunities. The project broke down into two major categories: building the images and hosting the images. 

Building the images required taking the base images provided by Microsoft and adding WinRM support to it. Without WinRM support, Vagrant will hang after the box starts attempting to interact with the virtual machine. Downloading and preparing the images was done with a combination of PhantomJS, Python, and the Python Virtualbox API. Each image was downloaded, booted, WinRM installed, and shutdown. The details of this process are beyond the scope of this post, but it's fair to say it wasn't the prettiest process and didn't lend itself to complete automation. From there, the boxes were repacked with `vagrant pack` and readied to be hosted.

For various reasons, I wanted to find a way to host the images in a way that would make it as easy as possible for clients to consume. This meant hosting it in a format understood by Vagrant. Thanks to a post by [@jonursenbach](https://twitter.com/jonursenbach) on how to host [Your own Vagrant Cloud](https://medium.com/@jonursenbach/your-own-vagrant-cloud-f077625c6ac8), I learned it is possible to host your private version of the public services like [VagrantCloud](http://vagrantcloud.com). 

A few google searches later led me to the [vagrant-catalog](https://github.com/vube/vagrant-catalog) project by [Ross Perkins](https://github.com/ross-p) at [vube](http://vubeology.com/). This is a PHP implementation of the structure required by Vagrant. I'm not a PHP or Apache afficianado so I was unsure of how to get this going. This is where a little Docker & Vagrant magic accelerated my process tremendously.

By using a pre-built docker image setup for Apache and PHP, [tutum/apache-php](https://registry.hub.docker.com/u/tutum/apache-php/), I was able to get setup quickly. It minimized my ramp-up to a few configuration changes specific to vagrant-catalog. A few hours later I had created [docker-vagrant-catalog](https://github.com/markhuber/docker-vagrant-catalog) and published it to the Docker Hub Registry as [markhuber/vagrant-catalog](https://registry.hub.docker.com/u/markhuber/vagrant-catalog/). 

```Dockerfile
FROM tutum/apache-php

    RUN apt-get update && apt-get install -yq git && rm -rf /var/lib/apt/lists/*
    
    RUN rm -rf /var/www/html && git clone https://github.com/vube/vagrant-catalog /var/www/html
    
    COPY 000-default.conf /etc/apache2/sites-available/000-default.conf

    WORKDIR /var/www/html
    RUN composer update
    RUN mv config.php.dist config.php
    RUN rm index.html
    
    EXPOSE 80
    WORKDIR /app
    CMD ["/run.sh"]
```

The workflow of connecting the Docker Registry to my GitHub repository meant every commit to master of my repo was automatically built into a Docker container. This was incredibly easy and quick to setup. I was able to test all of this with the Vagrant image [williamyeh/ubuntu-trusty-docker](https://vagrantcloud.com/williamyeh/boxes/ubuntu-trusty64-docker) from [@William-Yeh](https://github.com/William-Yeh). My testing setup can be found in my [modern-ie-vagrant-builder](https://github.com/markhuber/modern-ie-vagrant-builder) project [here](https://github.com/markhuber/modern-ie-vagrant-builder/tree/master/vagrant-catalog-server). By taking a portable approach with Docker and testing my changes in a Vagrant image, It was straight forward to move this setup to a EC2 hosted instance without any issue. 

More details about this workflow are available in this presentation slide [here](https://docs.google.com/presentation/d/1_p4epNSKS2a8NOdqM_pNab5Xa-H3hDwjnzVAt7GK3eQ/edit#slide=id.g9d50a47c9_0_0).
