# Instructions for installing LAMP on a KaliPi installation on Raspberry Pi 3B+ plus Concrete5


_Once KaliPi is installed, you will be logged in as root-_

* [Update Your Installation](#update-your-installation-and-get-some-basics-installed)
* [Remote Access](#set-yourself-up-with-remote-access-of-the-gui-desktop-with-vnc)  
* [Install Apache](#install-apache)
* [Install MariaDB](#install-mariadb)
* [Install PHP](#install-php)
* [Install NodeJS](#install-nodejs-and-less)
* [Install Concrete5](#install-concrete5)


## Update your installation and get some basics installed

### Expand the SD card to use all of it:
```sh
foo@bar:~$ ~/scripts/rpi-wiggle.sh
```

### Update kali:
```sh
foo@bar:~$ apt update
foo@bar:~$ apt upgrade
```

### Setup your user configuration:

Change root password:

```sh
foo@bar:~$ passwd root
```

Add a user:

```sh
foo@bar:~$ adduser <your_username>
```

Grant sudo privileges to your new user:

```sh
foo@bar:~$ usermod -aG sudo <your_username>
```

Switch to the new user:

```sh
foo@bar:~$ su <your_username>
```

### Setup git:

Configure:

```sh
foo@bar:~$ git config --global user.name <your_username>
foo@bar:~$ git config --global user.email <your_email>
```

Confirm what was saved:

```sh
foo@bar:~$ git config --list
```

**You can optionally install the kalipi-config from RPi-Tweaks but you must do as root, not sudo**

https://github.com/Re4son/RPi-Tweaks/blob/master/kalipi-config/README.md

Go through the setup and change localization, wifi, etc.  **do not change the NETWORK OPTIONS - HOSTNAME** it interferes with things later, OR you can add the following into /etc/hosts to match your /etc/hostname file

```sh
127.0.1.1	<your hostname>
```

**make sure you use 127.0.1.1 and not 127.0.0.1**

Enter into the config screen:

```sh
foo@bar:~$ kalipi-config
```


## Set yourself up with remote access of the GUI desktop with VNC  

**_xfce and Firefox should have been installed with KaliPi but if not, run the following._**

```sh
foo@bar:~$ apt install xfce4
foo@bar:~$sudo apt install firefox-esr
```




### Install VNC server:

```sh
foo@bar:~$ apt install tightvncserver
```

**Could add VNC user for security:**
```sh
foo@bar:~$ sudo adduser vnc
foo@bar:~$ sudo gpasswd -a vnc sudo
```

you must start the server each time __or__  - [Set VNC to run at startup](#set-vnc-to-run-at-startup)

```sh
foo@bar:~$ vncserver
```

You can kill the process with the following (the <display#> shows when you start the server)

```sh
foo@bar:~$ vncserver -kill :<display#>
```

Can use port forwarding for a secure connection.  In Termius, create a new port forwarding rule:


```sh
---LOCAL TAB---

Label:<whatever label you want>
Host:<choose the connection you use to access your vm>
Port from:<5902>
Destination:<localhost>
Port to:<5901> /*should be the same as what you default to when starting VNC*/
```

To use the secure connection, enable the ssh connection and also enable the port forwarding rule, then go to VNC and created a new connection with the address of <localhost:5902> and same password.  It should connect securely, even though you will still get a not secure warning.


### Set VNC to run at startup:

Next, we'll set up the VNC server as a systemd service so we can start, stop, and restart it as needed, like any other service. This will also ensure that VNC starts up when your server reboots.

First, create a new unit file called /etc/systemd/system/vncserver@.service using your favorite text editor:

```sh
foo@bar:~$ sudo vim /etc/systemd/system/vncserver@.service
```

The @ symbol at the end of the name will let us pass in an argument we can use in the service configuration. We'll use this to specify the VNC display port we want to use when we manage the service.

Add the following lines to the file. Be sure to change the value of User, Group, WorkingDirectory, and the username in the value of PIDFILE to match your username:


```sh
[Unit]
Description=Start TightVNC server at startup
After=syslog.target network.target

[Service]
Type=forking
User=<your_username>
Group=sudo
WorkingDirectory=/home/<your_username>

PIDFile=/home/<your_username>/.vnc/%H:%i.pid
ExecStartPre=-/usr/bin/vncserver -kill :%i > /dev/null 2>&1
ExecStart=/usr/bin/vncserver -depth 24 -geometry 1366x1024 :%i
ExecStop=/usr/bin/vncserver -kill :%i

[Install]
WantedBy=multi-user.target
```



The ExecStartPre command stops VNC if it's already running. The ExecStart command starts VNC and sets the color depth to 24-bit color with a resolution of 1366x1024. You can modify these startup options as well to meet your needs.

Save and close the file.

Next, make the system aware of the new unit file.

```sh
foo@bar:~$ sudo systemctl daemon-reload
```

Enable the unit file.

```sh
foo@bar:~$ sudo systemctl enable vncserver@1.service
```

The 1 following the @ sign signifies which display number the service should appear over, in this case the default :1 as was discussed in Step 2..

Stop the current instance of the VNC server if it's still running.

```sh
foo@bar:~$ vncserver -kill :1
```

Then start it as you would start any other systemd service.

```sh
foo@bar:~$ sudo systemctl start vncserver@1
```

You can verify that it started with this command:

```sh
foo@bar:~$ sudo systemctl status vncserver@1
```

If it started correctly, the output should look like this:

```sh
Output
 vncserver@1.service - Start TightVNC server at startup
 Loaded: loaded (/etc/systemd/system/vncserver@.service; enabled; vendor preset: enabled)
 Active: active (running) since Wed 2018-09-05 16:47:40 UTC; 3s ago
 Process: 4977 ExecStart=/usr/bin/vncserver -depth 24 -geometry 1280x800 :1 (code=exited, status=0/SUCCESS)
 Process: 4971 ExecStartPre=/usr/bin/vncserver -kill :1 > /dev/null 2>&1 (code=exited, status=0/SUCCESS)
 Main PID: 4987 (Xtightvnc)

...
Your VNC server will now be available when you reboot the machine.
```

**there may be a way to better secure the tunnel with -localhost**



## Install Apache

```sh
foo@bar:~$ apt install apache2
```

Allow Apache to start on reboot:

```sh
foo@bar:~$ sudo systemctl enable apache2
```

Modify /etc/apache2/apache.conf and add the following:

```sh
<Directory /home/<your_username>/websites>
	Options Indexes FollowSymLinks
	AllowOverride None
	Require all granted
</Directory>
```

and also include virtual host configuration by uncommenting:

```sh
IncludeOptional sites-enabled/*.conf
```

SAVE THE FILE, THEN

Change directory to sites-available

```sh
foo@bar:~$ cd sites-available
```

Copy 000-default.conf to a new virtual host configuration:

```sh
foo@bar:~$ cp 000-default.conf <eg. localhost.conf or any other name of your virtual site such as myVirtualSite.conf>
```

Modify /etc/apache2/sites-available/000-default.conf to include the new root path:

```sh
foo@bar:~$ DocumentRoot /home/<your_username>/websites 
```

Enable each virtual site created:

```sh
foo@bar:~$ sudo a2ensite <virtualhost.conf>
```

Chmod command to set permissions:

```sh
foo@bar:~$ chmod -R 755 /home/<your_username>/websites
```

You should also set permissions (???I think this sets permissions for virtual host sites???):

```sh
foo@bar:~$ sudo chown www-data:www-data /var/www/html/ -R
```

Reload and restart apache2:

```sh
foo@bar:~$ sudo systemctl reload apache2
foo@bar:~$ sudo systemctl restart apache2
```

Think about installing ufw firewall???

After apache2 installs you should see the Apache2 Debian page when accessing your ip address in a browser.
You can also access "localhost" through VNC when using a web browser.







## Install MariaDB

```sh
foo@bar:~$ sudo apt install mariadb-server
foo@bar:~$ sudo systemctl start mysql
foo@bar:~$ sudo mysql_secure_installation
```

USE NOTHING FOR THE ROOT PASSWORD ON FIRST QUESTION!!! JUST USE ENTER KEY!!!
THEN SET A NEW ROOT PASSWORD.  ALL OTHER OPTIONS CAN BE DEFAULT.

```sh
foo@bar:~$ sudo mariadb
MariaDB [(none)]> GRANT ALL ON *.* TO '<your_username>'@'localhost' IDENTIFIED BY '<password>' WITH GRANT OPTION;
```

### I set <your_username> above as my user in kali with the associated password

```sh
MariaDB [(none)]> FLUSH PRIVILEGES;

MariaDB [(none)]> exit

foo@bar:~$ mariadb -u <admin> -p
```




## Install PHP

```sh
foo@bar:~$ sudo apt install php libapache2-mod-php php-mysql
```

### Install the following for Concrete5 dependencies:

```sh
foo@bar:~$ sudo apt install php-xml
foo@bar:~$ sudo apt install php-gd
foo@bar:~$ sudo apt install php-mbstring
foo@bar:~$ sudo apt install php7.3-zip   #verify your current php version to install
```

### Create a info.php file with vim:

```sh
foo@bar:~$ vim /home/<your_username>/<websites>/info.php
```

enter the following...

```sh
<?php
phpinfo();
?>
```

...and save it.  Look in your browser at the localhostip/info.php to test if php is working.

This php page will show the location of the php.ini file you need to modify below but **__delete the info.php file when done for security.__**

Also add the following to extensions in php.ini:
    
```sh
extension=dom
uncomment extension=mbstring
```

modify apache config to look for php files first

```sh
foo@bar:~$ sudo vim /etc/apache2/mods-enabled/dir.conf

foo@bar:~$ sudo systemctl restart apache2
```






## Install nodejs and less

```sh
foo@bar:~$ sudo apt install nodejs npm
```

if that doesnt work, try

```sh
foo@bar:~$ sudo curl -sL https://www.npmjs.com/install.sh | sudo bash -

###foo@bar:~$ sudo apt install nodejs
```


__also had to run this command after and not sure why.  There was a note at the end of the install script stating to use this install command.  Maybe use this first and the previous isn't needed.__

```sh
foo@bar:~$ sudo apt-get install -y nodejs    
```

check version

```sh
foo@bar:~$ nodejs -v
foo@bar:~$ npm -v

foo@bar:~$ sudo npm install less -g
```




## Install Concrete5 
(reference- https://www.howtoforge.com/tutorial/install-and-configure-concrete5-cms-on-debian-9/ ):

Install zip:

```sh
foo@bar:~$ sudo apt install unzip  #came installed with KaliPi
```

Download the latest zip file from Concrete5 site and unzip it into the websites directory:

```sh
foo@bar:~$ unzip <concrete5version.zip> -d /home/<your_username>/websites
```

Create a new database in mysql:

```sh
foo@bar:~$ mysql -u <your_username> -p
MariaDB [(none)]> CREATE DATABASE IF NOT EXISTS <new database>;
MariaDB [(none)]> GRANT ALL ON *.* TO '<your_username>'@'localhost' IDENTIFIED BY '<password>' WITH GRANT OPTION;
MariaDB [(none)]> FLUSH PRIVILEGES;
MariaDB [(none)]> exit
```

Set proper permissions:

```sh
foo@bar:~$ sudo chown -R www-data:www-data /home/<your_username>/websites/<sandbox or whatever directory you unzipped concrete5 to>
foo@bar:~$ sudo chmod -R 775 /home/<your_username>/websites/<sandbox or whatever directory as above>
```

Restart apache2:

```sh
foo@bar:~$ sudo systemctl restart apache2
```

Open a browser at localhost ip address and go to the index.html page within the newly installed concrete5 directory to install concrete5.  I used 192.168.1.22 for the web url and mariaDB database, user and password accordingly.

**use localhost as the server location.**

