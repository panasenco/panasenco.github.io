---
layout: post
title: Full Guide to Deploying Django
date: 2015-04-15
disqus: y
---

### Step 0. Install Debian 7 on your server and power it on.

### Step 1. Set miscelanneous settings on your new server.

1. If you rebuilt the server, use the command ``ssh-keygen -R XX.XX.XX.XX`` on
your personal computer to remove the old SSH authentication information. Then connect:

    ```
    ssh root@XX.XX.XX.XX
    ```
    
1. Set the hostname (replace [hostname] with your desired hostname, e.g. Oxygen, Mercury, etc.).

    ```
    echo "[hostname]" > /etc/hostname
    hostname -F /etc/hostname
    ```
    
    Update /etc/hosts to reflect the change.
    
    ```
    nano /etc/hosts
    ```
    
    Add the following line:
    
    ```
    XX.XX.XX.XX [lowercase_hostname].[sitename.com] [hostname]
    ```
    
    (replacing [hostname] with your chosen hostname and [sitename.com] with your website URL)
    
    Close the SSH connection and log in again to see the changes
    
1. Update your timezone using the timezone utility.

    ```
    dpkg-reconfigure tzdata
    ```
    
1. Update and upgrade all Debian packages. This might take a few minutes.
    
    ```
    apt-get update
    apt-get upgrade --show-upgraded
    ```

### Step 2. Create a non-root user

1. Create a new user

    ```
    adduser [username]
    ```

1. Add the user to the admin group

    ```
    usermod -a -G sudo [username]
    ```

### Step 3. Enable SSH-key-pair authentication.

1. *(On your desktop computer)* Generate a SSH key pair, if you haven't already.

    ```
    ssh-keygen -t rsa -b 2048 -f ~/.ssh/[sitename]_rsa
    ```

1. *(On your desktop computer)* Install ssh-copy-id if you don't have it.

    ```
    brew install ssh-copy-id
    ```

1. *(On your desktop computer)* Copy your public key to the server.
    
    ```
    ssh-copy-id -i ~/.ssh/[sitename]_rsa.pub [username]@XX.XX.XX.XX
    ```

1. *(On your desktop computer)* Edit your SSH config file to use that key pair to log onto that particular server from now on.
    
    Open the file:
    
    ```
    nano .ssh/config
    ```
    
    Add the following lines:
    
    ```
    Host XX.XX.XX.XX
     IdentitiyFile ~/.ssh/[sitename]_rsa

1. Log onto your server as the newly created admin user.
    
    ```
    ssh [username]@XX.XX.XX.XX
    ```
    
    > NOTE: If the SSH key pair was set up correctly, you should not be asked
    > for your password.

### Step 4. Disable password authentication on your server.
    
1. Open the global SSH config file on your server.

    ```
    sudo nano /etc/ssh/sshd_config
    ```
    
1. Find and change the following settings in the above file:
    
    ```
    PasswordAuthentication no
    PermitRootLogin no
    ```

1. Restart the SSH service.

    ```
    sudo service ssh restart
    ```

### Step 5. Set up a firewall for your server.

1. Create a firewall rules file.

    ```
    sudo nano /etc/iptables.firewall.rules
    ```

1. Paste the following into the rules file:

    ```
    *filter
    
    #  Allow all loopback (lo0) traffic and drop all traffic to 127/8 that doesn't use lo0
    -A INPUT -i lo -j ACCEPT
    -A INPUT -d 127.0.0.0/8 -j REJECT
    
    #  Accept all established inbound connections
    -A INPUT -m state --state ESTABLISHED,RELATED -j ACCEPT
    
    #  Allow all outbound traffic - you can modify this to only allow certain traffic
    -A OUTPUT -j ACCEPT
    
    #  Allow HTTP and HTTPS connections from anywhere (the normal ports for websites and SSL).
    -A INPUT -p tcp --dport 80 -j ACCEPT
    -A INPUT -p tcp --dport 443 -j ACCEPT
    
    #  Allow SSH connections
    #
    #  The -dport number should be the same port number you set in sshd_config
    #
    -A INPUT -p tcp -m state --state NEW --dport 22 -j ACCEPT
    
    #  Allow ping
    -A INPUT -p icmp --icmp-type echo-request -j ACCEPT
    
    #  Log iptables denied calls
    -A INPUT -m limit --limit 5/min -j LOG --log-prefix "iptables denied: " --log-level 7
    
    #  Drop all other inbound - default deny unless explicitly allowed policy
    -A INPUT -j DROP
    -A FORWARD -j DROP
    
    COMMIT
    ```

1. Activate the new firewall rules.

    ```
    sudo iptables-restore < /etc/iptables.firewall.rules
    ```

1. Create a new network script for the firewall.

    ```
    sudo nano /etc/network/if-pre-up.d/firewall
    ```

1. Paste the following into the network script:

    ```
    #!/bin/sh
    /sbin/iptables-restore < /etc/iptables.firewall.rules
    ```

1. Make the script an executable.

    ```
    sudo chmod +x /etc/network/if-pre-up.d/firewall
    ```

### Step 6. Install Fail2Ban.

1. Install Fail2Ban to prevent SSH dictionary attacks:

    ```
    sudo apt-get install fail2ban
    ```

    > That's it (that was an easy step)!

### Step 7. Install, configure, and secure Apache2.

1. Install Apache2.
    
    ```
    sudo apt-get install apache2
    ```
    
    > You should now see the "It works!" page when you access the server using its IP address through your browser.

1. Make a copy of the Apache settings file, just in case.

    ```
    sudo cp /etc/apache2/apache2.conf /etc/apache2/apache2.backup.conf
    ```

1. Edit Apache2 global settings to configure it for the Linode 1GB.
    
    ```
    sudo nano /etc/apache2/apache2.conf
    ```

    Find the KeepAlive setting and make sure it's set to Off.

    ```
    KeepAlive Off
    ```

    Find the servers/clients/requests settings and make sure they are set to the following:

    ```
    <IfModule mpm_prefork_module>
        StartServers             2
        MinSpareServers          6
        MaxSpareServers         12
        MaxClients              30
        MaxRequestsPerChild   3000
    </IfModule>
    ```

1. Install mod_security

    ```
    sudo apt-get install libapache2-modsecurity
    ```

    Enable the mod_security recommended base configuration.

    ```
    cd /etc/modsecurity
    sudo cp modsecurity.conf-recommended modsecurity.conf
    sudo nano modsecurity.conf
    ```
    
    Change the line ``SecRuleEngine DetectionOnly`` to ``SecRuleEngine On``.
    
1. Restart Apache to apply changes.

    ```
    sudo service apache2 restart
    ```

### Step 8. Install and secure MySQL server

1. Install MySQL

    ```
    sudo apt-get install mysql-server libmysqlclient-dev
    ``` 

1. Start the secure installation process

    ```
    mysql_secure_installation
    ```
    
    Answer 'yes' to everything except when the process tells you it's OK to answer no.

1. Disable mysql_history logging.

    Open your .bashrc file:
    
    ```
    nano ~/.bashrc
    ```
    
    Add the following lines at the end of the file:
    
    ```
    # Disable MySQL History
    export MYSQL_HISTFILE=/dev/null
    ```
    
    Close nano with Ctrl+X. Now update the current settings by running:
    
    ```
    source ~/.bashrc
    ```    

1. Create a blank production database.
    
    Enter the MySQL interactive prompt:
    
    ```
    mysql -u root -p
    ```

    Inside the MySQL client, create a new production database.
    Be sure to use the exact same database name, username, and password as you did in development.
    
    *(note the period after the database name, it's important to keep it there)*
    
    ```
    CREATE DATABASE [dbname];
    CREATE USER '[user]'@'localhost' IDENTIFIED BY '[password]';
    GRANT ALL PRIVILEGES ON [dbname].* TO '[user]'@'localhost' WITH GRANT OPTION;
    FLUSH PRIVILEGES;
    EXIT;
    ```

### Step 9. Restore your development database on your production server (if you need to).

If you did any work on your development database that you'd like to see in your
production database, this section will tell you how to copy your development database
to your server. If you are OK with starting with a clean database, you can skip this step.

1. *(On your personal computer)* Open a new Terminal window and work with your personal computer for a minute.
Start your MySQL server. On a Mac, you can use the following command:

    ```
    mysql.server start
    ```

1. *(On your personal computer)* You're going to make a "dump" of your development database. 
Here, [user] is the user you used during the development of your Django app and
[dbname] is the name of the database you used.
You will be prompted to enter the password for that user

    ```
    mysqldump -u [user] -p [dbname] > ~/dbdump.sql
    ```

1. *(On your personal computer)* Copy the database dump onto the Linux server using scp (Secure Copy).
Once again, replace [username] with your server username and XX.XX.XX.XX with your server's IP address.

    ```
    scp ~/dbdump.sql [username]@XX.XX.XX.XX:~/
    ```

1. *(Back to using SSH with the remote server)* Now go back to the window with
the SSH connection to your server. Make sure the database dump file is now on your server.

    ```
    ls ~
    ```
    
    You should see the dbdump.sql file in the output.

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
    
    > You should now have an exact copy of your development database set up on your production server.

### Step 10. Copy your Django application to your server.

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

1. *(On your personal computer)* Use rsync to transfer all site files (including static files and the Pip dependencies file) from your computer to the server.
This might take a while.

    ```
    rsync -au [/path/to/site/]* root@XX.XX.XX.XX:~/[sitename.com]/
    ```

1. *(Back to using SSH with the remote server)* The Django files are now on your server, in your user's home folder. We want to move them to /var/www/.

    ```
    sudo mv [sitename.com] /var/www/[sitename.com]
    ```

### Step 10ALT. Clone your Django app from your private GitHub repo.

1. Generate an SSH key pair on your server.
    
    ```
    ssh-keygen -t rsa -b 2048 -f ~/.ssh/id_rsa
    ```

1. Use cat to dump the public key's contents in your terminal window.
    
    ```
    cat ~/.ssh/id_rsa.pub
    ```
    
1. Copy the entire public key and activate it with your GitHub account.
    
    Copy the entire output of ``cat`` right from your terminal window. Then log
    into GitHub using your browser. Go to Personal settings > SSH keys.
    Click "Add SSH key". Create a title that mentions the hostname, IP, and
    username. Then paste the contents of the key into the Key field.
    
1. Authenticate with GitHub

    ```
    ssh -T git@github.com
    ```

1. Create a new git folder in your home folder.

    ```
    mkdir ~/git
    ```

1. Clone your Django project repository into your git folder
    
    ```
    git clone git@github.com:[github username]/[repo name].git ~/git/[repo name]
    ```

1. Use a server-side deployment script to deploy your site from its git folder.
Here are the contents of mine:
    
    ```
    # Sync the server with the GitHub site, deleting stale files
    sudo rsync -dvr ~/git/krafty/site/ /var/www/kraftyapp.com --exclude 'log' --delete-delay
    
    # Repair permissions of the site
    sudo chown -R www-data:www-data /var/www/kraftyapp.com
    find /var/www/kraftyapp.com -type d -print0 | sudo xargs -0 chmod 0775
    find /var/www/kraftyapp.com -type f -print0 | sudo xargs -0 chmod 0664
    
    # Restart the server
    sudo service apache2 restart
    ```
    
    Note that this script gives ownership of the site's directory to user
    www-data and group www-data. If you want your current user to have write
    access to the site, use the command ``sudo usermod -aG www-data [username]``

1. (Optional) For one-step deployment, create a local deployment script that
you can run from your local computer's command line. Here's mine:
    
    ```
    ssh -t user@XX.XX.XX.XX "cd ~/git/[repo] && git pull origin master && source ~/git/[repo]/scripts/server_deploy.sh"
    ```

1. Because our server_deploy.sh script uses sudo, this will ask us for a
password every time. To avoid that, do the following while logged in on the
remote server:

    Open the sudoers settings file for modification:
    
    ```
    sudo visudo
    ```
    
    Add the following line at the very end of the file:
    
    ```
    [user] ALL=(ALL) NOPASSWD:ALL
    ```
    
    Exit using Ctrl+X and save. Now using sudo will never require you to enter
    your password. This is convenient for fast, automatic builds.

### Step 11. Enable a Python Virtual Environment

1. Install Pip and virtualenvwrapper.

    ```
    sudo apt-get install python3-pip
    sudo pip-3.2 install virtualenvwrapper
    ```

1. Enable virtualenvwrapper.

    Edit your .bashrc file again:
    
    ```
    nano ~/.bashrc
    ```
    
    Add the following lines to the end of the file:
    
    ```
    # Setting up environment variables for virtualenvwrapper
    export VIRTUALENVWRAPPER_PYTHON=/usr/bin/python3.2
    export WORKON_HOME=$HOME/.virtualenvs
    export PROJECT_HOME=/var/www
    source /usr/local/bin/virtualenvwrapper.sh
    ```
    
    Ctrl+X out of nano and apply the changes using ``source ~/.bashrc``.

1. Create a new virtual environment for virtualenvwrapper.
    
    ```
    mkvirtualenv [sitename]
    ```
    
    > The new environment is automatically activated.
    > You can re-activate it in the future by running ``workon [sitename]``.

### Step 12. Set up your Django application.

1. Now you can install the required Pip packages (including Django itself) inside
your virtual environment using the dependency list you generated earlier.

    Note that you'll be using ``pip`` now, not ``pip-3.2``. This is important because
    ``pip`` points to the pip in your virtual environment, and ``pip-3.2`` points
    to the pip installed in /usr/bin. If you install the dependency list using
    ``pip-3.2``, it will be installed on your entire server, not in your virtual environment.
    
    ```
    cd /var/www/[sitename.com]
    pip install -r pip_dependencies.txt
    ```
    
    
1. Make sure Django and all packages were installed correctly by running the Django development server on your Linux server.
    
    Stop the Apache server first.
    
    ```
    sudo service apache2 stop
    ```
    
    Get the path to the virtual environment's python so you can run it as root:
    
    ```
    which python
    ```
    
    This should give you something like ``/home/[user]/.virtualenvs/[sitename]/bin/python``.
    Copy and paste that path in place of [virtual python] below.
    Note the use of settings.local. I use multiple settings files inside a settings
    directory to manage my settings. You should adopt a similar approach to allow for
    fast switching between settings files.
    
    ```
    cd /var/www/[sitename.com]/[projectname]
    sudo [virtual python] manage.py runserver 0.0.0.0:80 --settings settings.local
    ```

    > At this point, you should be able to access your server through its IP address in your browser window and see your Django app. Your Django app is probably going to be in debug mode.

1. Now it's time to turn debug mode off. If you use multiple settings files,
it's as easy as changing a word in the manage.py command.
    
    If you don't use multiple settings files, be sure the following are set in
    your settings.py:
    
    ```
    DEBUG = False
    TEMPLATE_DEBUG = False
    ALLOWED_HOSTS = ['.[sitename.com]', '.XX.XX.XX.XX']
    import sys
    sys.path.append('/var/www/[sitename.com]/[projectname]')
    ```
    
    where XX.XX.XX.XX is your server's IP address.

1. Run the Django development server with debug mode turned off.
    
    Use of the --insecure keyword allows Django to serve your static files through its development server when debug is turned off.
    
    ```
    cd /var/www/[sitename].com/[projectname]
    sudo [virtual python] manage.py runserver 0.0.0.0:80 --settings settings.production --insecure
    ```
    
    > At this point, the Django app should still be accessible through your server's IP address, but now debug mode is turned off.

### Step 13. Install Mod-WSGI and configure a VirtualHost file for your site.

1. Disable the default Apache virtual host

    ```
    sudo a2dissite *default
    ```

1. Create a new virtual host configuration file.
    
    ```
    sudo nano /etc/apache2/sites-available/[sitename.com]
    ```

1. Paste the following into the configuration file:

    ```
    <VirtualHost *:80>
        # Admin email, Server Name (domain name), and any aliases
        ServerAdmin [your@email.com]
        ServerName  [www.sitename.com]
        ServerAlias [sitename.com]
        
        # Log file locations
        LogLevel warn
        ErrorLog  /var/www/[sitename.com]/log/error.log
        CustomLog /var/www/[sitename.com]/log/access.log combined
        
        # WSGI interface settings
        WSGIScriptAlias / /var/www/[sitename.com]/[projectname]/[projectname]/wsgi.py
        WSGIDaemonProcess processes=2 threads=15
        <Directory /var/www/[sitename.com]/[projectname]/[projectname]>
            <Files wsgi.py>
                Order allow,deny
                Allow from all
            </Files>
        </Directory>
        
        # Regular static files
        Alias /static /var/www/[sitename.com]/static/static
        <Directory /var/www/[sitename.com]/static/static>
            Order allow,deny
            Allow from all
        </Directory>
        
        # Expected static files (favicon.ico, sitemap.xml, robots.txt etc.)
        AliasMatch ^/([^/]+)\.(ico|png|xml|txt|json)$ /var/www/[sitename.com]/static/root/$1.$2
        <Directory /var/www/[sitename.com]/static/root>
            Order allow,deny
            Allow from all
        </Directory>
    </VirtualHost>
    ```
    
    Note the use of AliasMatch at the end. That serves all requests for root files
    ending in .ico, .png, .xml, and .json to a special static directory called
    root - that's where favicons, browser configurations, app configurations,
    and sitemaps live. Favicons were generated with http://realfavicongenerator.net

1. Enable the new virtual host.
    
    ```
    sudo a2ensite kraftyapp.com
    ```

### Step 14. Install and configure Mod-WSGI for Apache2.

1. Install Python3 Mod-WSGI for Apache2.

    ```
    sudo apt-get install libapache2-mod-wsgi-py3
    ```

1. Make sure your wsgi.py is configured properly.
    
    ```
    nano /var/www/kraftyapp.com/kraftyapp/kraftyapp/wsgi.py
    ```
    
    This is what my wsgi.py file looks like:
    
    ```
    # Activate the correct virtual environment on the server.
    activate_env='/home/krafty/.virtualenvs/krafty/bin/activate_this.py'
    exec(open(activate_env).read(), dict(__file__=activate_env))
    
    import os, sys
    # Add the Django project's path to the python path.
    sys.path.append('/var/www/kraftyapp.com/kraftyapp')
    # Set the settings file to be used in production.
    os.environ.setdefault('DJANGO_SETTINGS_MODULE', 'settings.production')
    
    # Start the WSGI aplication.
    from django.core.wsgi import get_wsgi_application
    application = get_wsgi_application()
    ```
    
    Yours can be different. Make sure you activate the virtual environment that
    you are working in. Make sure your Django project is in your Python path.

1. Start (or reload) Apache2.
    
    ```
    sudo service apache2 start
    ```
    
    Or if it's already running:
    
    ```
    sudo service apache2 reload
    ```
    
    > You can now navigate to your site's IP address and see your Django app!
    
    