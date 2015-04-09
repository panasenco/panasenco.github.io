---
layout: post
title: Quick and Dirty Django Deployment
date: 2015-04-08
---

Watch the video at:

> ##WARNING
> This tutorial is meant to take you through the bare minimum number of steps to set up your own Django production server.
>
> This means **skipping over** steps that deal with **safety, security, and general production-readiness**.
>
> After you follow this tutorial, you will have to **start over with a clean Linux install** and follow other tutorials to set up your server in a secure and optimized way.

##Deploying your own quick and dirty Django server:

1. Log into your new server as root over SSH (Secure Shell Connection).
    XX.XX.XX.XX is the IP address of your server.
    
    ```
    ssh root@XX.XX.XX.XX
    ```

1. Update and upgrade all Debian packages (necessary even for a quick-and-dirty install).
    
    ```
    apt-get update
    apt-get upgrade
    ```

1. Install the Apache 2 server.
    
    ```
    apt-get install apache2
    ```
  
1. 