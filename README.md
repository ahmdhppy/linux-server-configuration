# Linux Server Configuration
 ## Project  Description:

This is the project for the Udacity Full Stack Nanodegree., called "Linux Server Configuration"
The Linux Server Configuration project involves taking a baseline installation of Linux on a virtual machine and preparing it to host web applications. This includes installing updates, securing the server from attacks, and installing / configuring web and database servers. 

### Server Information:

- IP address: 159.65.121.225
- URL : http://www.item-catalog.info/
- To connect: ssh -p 2200 -i item_catalog grader@159.65.121.225 


###Configurations:
1. Update all currently installed packages.
    ```bash
    #Update list of available packages.
    apt update

    #Upgrade the system by installing/upgrading packages.
    apt upgrade
    ```
2. Add grader user and give it sudo access.
    ```bash
    adduser grader
    usermod -aG sudo grader
    ```
3. Create authorized_keys file for grader user.
    ```bash
    #Change the permissions for .ssh folder and authorized_keys file
    chmod 700 /home/grader/.ssh
    chmod 644 /home/grader/.ssh/authorized_keys

    #Change the owner of .ssh folder and authorized_keys file to grader user
    chown -R grader:grader /home/grader/.ssh
    ```
4. Configure the key-based authentication for grader user.
    - Generate an encryption key on your local machine with.
        ```bash
        ssh-keygen -t rsa -b 4096 -f ./item_catalog
        ```
   
    - Copy the content of the *item_catalog.pub* file from your local machine to the */home/grader/.ssh/authorized_keys* file you just created on the remote.
    
    - Now you are able to log into the remote VM through ssh with the following command:
        ```bash
        ssh -i ~/.ssh/item_catalog grader@159.65.121.225
        ```
3. Secure shared memory.
    ```bash
    echo "tmpfs /run/shm tmpfs defaults,noexec,nosuid 0 0" >> /etc/fstab
    ```
4. Configure local timezone to UTC.
    ```bash
    timedatectl set-timezone UTC
    ```
5. SSH Configuration.
    ```bash
    #Enable SSH Login For 'grader' User Only.
    echo "AllowUsers grader" >> /etc/ssh/sshd_config
    
    #Disable remote root login.
    echo "PermitRootLogin no" >> /etc/ssh/sshd_config
    
    #Disallow SSH password authentication.
    echo "PasswordAuthentication no" >> /etc/ssh/sshd_config
    
    #Change the port number that sshd listens on.
    echo "Port 2200" >> /etc/ssh/sshd_config
    
    #Change the protocol versions.
    echo "Protocol 2" >> /etc/ssh/sshd_config

    #Create .ssh directory and  authorized_keys file for grader user.
    mkdir /home/grader/.ssh
    touch /home/grader/.ssh/authorized_keys

    

    #Restart Ssh Service
    service ssh restart 
    ```
6. Configure the uncomplicated firewall.
    ```bash
    #Deny default incoming connections.
    ufw default deny incoming

    #allow default outgoing connections.
    ufw default allow outgoing

    #Allow SSH connections on port 2200.
    ufw allow 2200/tcp
    
    #Allow HTTP connections on port 80.
    ufw allow 80/tcp
    
    #Allow NTP connections on port 123.
    ufw allow 123/udp
    
    #Enable the firewall.
    ufw enable

    #Check firewall status.
    ufw status
    ```
7. Set up automatic updates packages:
    ``` bash
    #Install unattended-upgrades package.
    apt install unattended-upgrades

    #Add some configuration to '/etc/apt/apt.conf.d/20auto-upgrades
    echo "APT::Periodic::Update-Package-Lists "1";" > /etc/apt/apt.conf.d/20auto-upgrades
    echo "APT::Periodic::Download-Upgradeable-Packages "1";" >> /etc/apt/apt.conf.d/20auto-upgrades
    echo "APT::Periodic::AutocleanInterval "7";" >> /etc/apt/apt.conf.d/20auto-upgrades
    echo "APT::Periodic::Unattended-Upgrade "1";" >> /etc/apt/apt.conf.d/20auto-upgrades

    ```


###Install required packages Apache2, mod_wsgi, python, git and PostgreSQL:
```bash
apt install apache2 libapache2-mod-wsgi python-dev postgresql python2.7 python-pip postgresql-server-dev-all git -y
```

- The following error raised when restart the apache.
    ```
     AH00558: apache2: Could not reliably determine the server's fully qualified domain name, using 127.0.0.1. Set the 'ServerName' directive globally to suppress this message.`

    ```
- Solve above issue by execut the follwing command and restart apache:
    ```bash
    echo "ServerName localhost" >> /etc/apache2/apache2.conf
    ```


###Analyse system LOG files - LogWatch:
* Install LogWatch.
    ```bash
    #Install LogWatch 
    apt install logwatch libdate-manip-perl

    #To view logwatch output use less:
    logwatch | less
    ``` 

###Configure postgres:
* Switch to default user postgres.
    ```bash
    sudo su - postgres
    ```
* Open the interactive terminal for working with postgres.
    ```bash
    psql
    ```
* Create user `item`.
    ```sql
    CREATE USER item WITH PASSWORD 'STRONG PASSWORD';
    ```
* Give the `item` user privileges to create database.
    ```sql
    ALTER USER item CREATEDB;
    ```
* Revoke all privileges from public.
    ```sql
    REVOKE ALL ON SCHEMA public FROM public;
    ```
* Give privileges for `item` user on public schema.
    ```sql
    GRANT ALL ON SCHEMA public TO item;
    ```

###Deploy Flask Application:
* Log into the remote VM through ssh with the following command:
    ```bash
    ssh -i ~/.ssh/item_catalog grader@159.65.121.225
    ```
* Create a new directory under */home/grader* with name  'item-catalog-project'.
    ```bash
    mkdir item-catalog-project
    ```
* Clone the item-catalog project from github under created directory.
    ```bash
    cd item-catalog-project
    git clone https://github.com/ahmdhppy/item-catalog.git
    ```
* Install the required python packages.
    ```bash
    sudo pip2 install -r ./item-catalog/requirements.txt
    ```
* Craete database and insert demo data.
    ```bash
    #Create database
    python2.7 ./item-catalog/create_db.py -c ./item-catalog/config.conf

    #Insert demo data
    python2.7 ./item-catalog/insert_data.py -c ./item-catalog/config.conf
    ```

* Create a new file under */home/grader/item-catalog-project* called app.wsgi
    ```bash 
    touch app.wsgi
    ```
* Add following line to app.wsgi file and save it.
    ```
    #! /usr/bin/python2.7

    import logging
    import sys
    sys.argv.extend(['-c', '/home/grader/item-catalog-project/item-catalog/config.conf'])
    logging.basicConfig(stream=sys.stderr)
    sys.path.insert(0, '/home/grader/item-catalog-project/item-catalog')
    from app import app as application
    from routes.routes import *

    ```
* Configure apache virtual hosts.
    - Open `/etc/apache2/sites-enabled/000-default.conf` with sudo by nano and delete everything in this file then add the follwing lins and save it.
    ```
    <VirtualHost *:80>
        ServerName localhost
        WSGIScriptAlias /  /home/grader/item-catalog-project/app.wsgi
        <Directory /home/grader/item-catalog-project>
            Options FollowSymLinks
            AllowOverride None
            Require all granted
        </Directory>
        ErrorLog ${APACHE_LOG_DIR}/error.log
        LogLevel warn
        CustomLog ${APACHE_LOG_DIR}/access.log combined
    </VirtualHost>

    ```
* Configure Item Catalog project by edit the follwing configuration file.
    - /home/grader/item-catalog-project/item-catalog/config.conf
* Restart the apache server.
    ```bash
    sudo apachectl restart
    ```

#Restart apache to launch the app


## Resources Used
[https://www.codementor.io/abhishake/minimal-apache-configuration-for-deploying-a-flask-app-ubuntu-18-04-phu50a7ft](https://www.codementor.io/abhishake/minimal-apache-configuration-for-deploying-a-flask-app-ubuntu-18-04-phu50a7ft)
[https://securitytrails.com/blog/mitigating-ssh-based-attacks-top-15-best-security-practices](https://securitytrails.com/blog/mitigating-ssh-based-attacks-top-15-best-security-practices)
[https://www.lifewire.com/harden-ubuntu-server-security-4178243](https://www.lifewire.com/harden-ubuntu-server-security-4178243)
[https://libre-software.net/ubuntu-automatic-updates/](https://libre-software.net/ubuntu-automatic-updates/)


