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

---

###Step 1. Log into your server and install Apache 2.

1. *(In your personal computer's Terminal)* Log into your new server as root over SSH (Secure Shell Connection).
    XX.XX.XX.XX is the IP address of your server.
    Type in the root password when prompted.
    
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
  
1. Open Apache's error log (SUPER useful)
    
    ```
    tail -f /var/log/apache2/error.log
    ```

> At this point, you can visit your site's IP address in your browser and see the
> default Apache installation page. This means that Apache has been installed
> successfully. If you don't see anything, check the error log to see what the
> problem is (and if you can fix it).

---

###Step 2. Install Mod-WSGI for Python3 and run a simple app.

1. Install Mod-WSGI for Apache 2 (this will install Python 3 automtically).
    
    ```
    apt-get install libapache2-mod-wsgi-py3
    ```

1. Create a new sample WSGI application in /var/www/wsgi-scripts/myapp.wsgi.
    
    ```
    cd /var/www
    mkdir wsgi-scripts
    cd wsgi-scripts
    nano myapp.wsgi
    ```

1. Paste the following into myapp.wsgi:

    ```
    def application(environ, start_response):
        status = '200 OK'
        output = 'Hello World!'
    
        response_headers = [('Content-type', 'text/plain'),
                            ('Content-Length', str(len(output)))]
        start_response(status, response_headers)
    
        return [output]
    ```

1. Open the default VirtualHost configuration file for editing and scroll all the way down.
    
    ```
    sudo nano /etc/apache2/sites-available/default
    ```

1. To make our sample script work, we need to set up a script alias, a new daemon process, and a directory directive.
Place them at the bottom of the default configuration file, right before the </VirtualHost> closing tag.
Replace example.com with your own site name.
    
    ```
    WSGIScriptAlias / /var/www/wsgi-scripts/myapp.wsgi
    WSGIDaemonProcess example.com processes=2 threads=15 display-name=%{GROUP}
    WSGIProcessGroup example.com
    <Directory /var/www/wsgi-scripts>
        Order allow,deny
        Allow from all
    </Directory>
    ```

1. Restart the Apache server to apply the changes. This is another useful command. When in doubt, restart the server.
    
    Then, open the error log again to track possible errors.
    
    ```
    service apache2 restart
    tail -f /var/log/apache2/error.log
    ```
> You can now access your server through your browser using its IP address.
> If WSGI was installed correctly, you'll see "Hello World". If not, check the error log.

> To make sure the WSGI daemon processes work, edit the myapp.wsgi file and replace "Hello World" with some other string, like "Hello Universe".
> If the daemon processes are working correctly, you should be able to refresh the site and see the new string - without restarting your Apache server.

---

###Step 3. Install MySQL and restore your development database.

1. *(In your personal computer's Terminal)* Make a "dump" of your development database.
Here, [user] is the user you used during the development of your Django app and [dbname] is the name of the database you used.
You will be prompted to enter the password for that user

    ```
    mysql dump -u [user] -p [dbname] > ~/dbdump.sql

1. *(In your personal computer's Terminal)* Copy the database dump onto the Linux server using scp (Secure Copy).
Once again, replace XX.XX.XX.XX with your server's IP address.

    ```
    scp ~/dbdump.sql root@XX.XX.XX.XX:~/
    ```

1. *(Back to )*Install MySQL

    ```
    apt-get install mysql-server libmysqlclient-dev
    ```

1. 

