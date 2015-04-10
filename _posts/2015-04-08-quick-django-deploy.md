---
layout: post
title: Quick and Dirty Django Deployment
date: 2015-04-08
disqus: y
---

[Click here to watch the video on Youtube](http://youtube.com)

> ## WARNING
> This tutorial is meant to take you through the bare minimum number of steps to set up your own Django production server.
>
> This means **skipping over** steps that deal with **safety, security, and general production-readiness**.
>
> After you follow this tutorial, you will have to **start over with a clean Linux install** and follow other tutorials to set up your server in a secure and optimized way.

---

### Step 1. Log into your server and install Apache 2.

1. *(On your personal computer)* Log into your new server as root over SSH (Secure Shell Connection).
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

### Step 2. Install Mod-WSGI for Python3 and run a simple app.

1. Install Mod-WSGI for Apache 2 (this will install Python 3 automatically).
    
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

    Paste the following into myapp.wsgi:

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
    nano /etc/apache2/sites-available/default
    ```

    To make our sample script work, we need to set up a script alias, a new daemon process, and a directory directive.
Place them at the bottom of the default configuration file, right before the ``</VirtualHost>`` closing tag.
Replace [sitename].com with your own site name.
    
    ```
    WSGIScriptAlias / /var/www/wsgi-scripts/myapp.wsgi
    WSGIDaemonProcess [sitename].com processes=2 threads=15 display-name=%{GROUP}
    WSGIProcessGroup [sitename].com
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
    > 
    > To make sure the WSGI daemon processes work, edit the myapp.wsgi file and replace "Hello World" with some other string, like "Hello Universe".
    > If the daemon processes are working correctly, you should be able to refresh the site and see the new string - without restarting your Apache server.

---

### Step 3. Install MySQL and duplicate your development database.

1. *(On your personal computer)* Make a "dump" of your development database.
Here, [user] is the user you used during the development of your Django app and [dbname] is the name of the database you used.
You will be prompted to enter the password for that user

    ```
    mysql dump -u [user] -p [dbname] > ~/dbdump.sql

1. *(On your personal computer)* Copy the database dump onto the Linux server using scp (Secure Copy).
Once again, replace XX.XX.XX.XX with your server's IP address.

    ```
    scp ~/dbdump.sql root@XX.XX.XX.XX:~/
    ```

1. *(Back to using SSH with the remote server)* Install MySQL on your server.

    ```
    apt-get install mysql-server libmysqlclient-dev
    ```

1. Enter the MySQL client interactive prompt.

    ```
    mysql -u root -p
    ```

    Inside the MySQL client, create a blank copy of your development database with your development user.
    
    ```
    CREATE DATABASE [dbname];
    CREATE USER '[user]'@'localhost' IDENTIFIED BY '[password]';
    GRANT ALL PRIVILEGES ON [dbname].* TO '[user]'@'localhost' WITH GRANT OPTION;
    FLUSH PRIVILEGES;
    EXIT
    ```

1. Restore the development database from the dump.
    
    ```
    mysql -u [user] -p [dbname] < ~/dbdump.sql
    ```

1. To make sure the restoration went OK, re-enter the interactive prompt.
    
    ```
    mysql -u [user] -p
    ```
    
    Run the following commands to browse through the MySQL databases and tables.
    
    ```
    SHOW DATABASES;
    USE [dbname];
    SHOW TABLES;
    SELECT * FROM [tablename];
    ```
    
    > You should now have an exact copy of your development database set up on your server.

---
### Step 4. Run Your Django Application On Your Server.

1. *(On your personal computer)* Make and apply any necessary migrations, collect all static files, and test that the development server works.

    ```
    cd [path/to/django]
    python3 manage.py makemigrations
    python3 manage.py syncdb
    python3 manage.py collectstatic
    python3 manage.py runserver
    ```

1. *(On your personal computer)* Use pip to make a list of pip dependencies inside your site's root

    ```
    cd [path/to/site]
    pip3 freeze > pip_dependencies.txt
    ```

1. *(On your personal computer) Use rsync to transfer all site files (including static files and the Pip dependencies file) from your computer to the server.
This might take a while.

    ```
    rsync -au [/path/to/site/]* root@45.33.46.81:/var/www/[sitename].com
    ```
    
1. *(Back to using SSH with the remote server)* The Django files are now on your server, but none of the required Pip packages (including Django itself) are installed.
Install them using the dependency list you generated earlier.

    ```
    cd /var/www/[sitename].com
    pip3 install -r pip_dependencies.txt
    ```

1. Make sure Django and all packages were installed correctly by running the Django development server on your Linux server.
You have to stop the Apache server first.
    
    ```
    service apache2 stop
    cd /var/www/[sitename].com/[projectname]
    python3 manage.py runserver 0.0.0.0:80
    ```

    > At this point, you should be able to access your server through its IP address in your browser window and see your Django app (in debug mode).

1. Now it's time to turn debug mode off. Open the Django settings.py file on your server.
    
    ```
    cd /var/www/[sitename].com/[projectname]/[projectname]
    nano settings.py
    ```
    
    Edit the following settings:
    
    ```
    DEBUG = False
    TEMPLATE_DEBUG = False
    ALLOWED_HOSTS = ['*']
    ```
    
    Also make sure the static files are still being served in the same way when debug mode is turned off.

1. Run the Django development server with debug mode turned off.
    
    ```
    cd /var/www/[sitename].com/[projectname]
    python3 manage.py runserver 0.0.0.0:80 --insecure
    ```
    
    > At this point, the Django app should still be accessible through your server's IP address, but now debug mode is turned off.

---
### Step 5. Link everything together using Mod-WSGI.

1. Open your project's wsgi.py file.
    
    ```
    cd /var/www/[sitename].com/[projectname]/[projectname]
    nano wsgi.py
    ```
    
    Add the following lines in the beginning of your wsgi.py file:
    
    ```
    import sys
    sys.path.append('/var/www/kraftyapp.com/kraftyapp')
    sys.path.append('/usr/local/lib/python3.4/site-packages')
    ```
    
    > Running python3 wsgi.py now shouldn't produce any errors.
    
1. Go back to editing your Apache2 default configuration file.
    
    ```
    nano /etc/apache2/sites-available/default
    ```
    
    Edit the WSGI settings we created above to point to your project's wsgi.py file.
    
    ```
    WSGIScriptAlias / /var/www/[sitename.com]/[projectname]/[projectname]/wsgi.py
    WSGIDaemonProcess [sitename.com] processes=2 threads=15 display-name=%{GROUP}
    WSGIProcessGroup [sitename.com]
    <Directory /var/www/[sitename.com]/[projectname]/[projectname]>
        Order allow,deny
        Allow from all
    </Directory>
    ```

1. In the same default config file, add a directory directive to serve static scripts.

    ```
    Alias /static /var/www/[sitename.com]/static
    <Directory /var/www/[sitename.com]/static>
            Require all granted
    </Directory>
    ```

1. Start the Apache 2 server again, keeping an eye on the error log.

    ```
    service apache2 start
    tail -f /var/log/apache2/error.log
    ```
    
    > Your Django app served through your Apache server should now be available through your server's IP address.

---

## Congratulations!

Your Django app is now accessible through your server.
Now that you know how to do it the quick and dirty way, follow these tutorials to do it the right way:

* [Set up your Django project using virtualenvwrapper and source control](http://www.jeffknupp.com/blog/2013/12/18/starting-a-django-16-project-the-right-way/)
* [Set a hostname and a timezone](https://www.linode.com/docs/getting-started#setting-the-hostname)
* [Create a non-root user, use SSH key-pair authentication, and set up a firewall](https://www.linode.com/docs/security/securing-your-server/)
* [Improve the security of your MySQL database](https://dev.mysql.com/doc/refman/5.0/en/mysql-secure-installation.html)
* [Go through the deployment checklist to make sure your Django app is ready to deploy](https://docs.djangoproject.com/en/1.8/howto/deployment/checklist/)
* [Set up mod_wsgi and use a different server to serve static files](https://docs.djangoproject.com/en/1.7/howto/deployment/wsgi/modwsgi/)

## Please give me feedback!

Did you like this tutorial? Do you think anything I suggested is horribly wrong? Leave a comment below and let me know!