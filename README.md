# PHP-Nginx-MySQL-Setup

A step-by-step guide to setting up a server for hosting PHP applications with Nginx and MySQL. Includes configuration instructions for Nginx, PHP-FPM, and MySQL, along with optimisation tips.

---

# Set Up and Secure a VPS 

I prefer using Ubuntu 24.04 LTS for setting up and securing services due to its stability and long-term support. As an LTS release, it receives security updates and critical patches for five years, ensuring a reliable environment for production servers. While other distributions may offer cutting-edge features, Ubuntu LTS releases focus on providing stable and well-tested software, minimizing the risk of bugs and conflicts, making it an ideal choice for hosting PHP applications with Nginx and MySQL.

If I am settings up the server for a client first thing I like to do is set a host name!

`hostnamectl hostname host.example.com`

Then point the user sub domain to server IP so I can SSH user service using `ssh root@host.example.com`

## Installing Software Updates

Even though you've just provisioned your new server, some software packages may already be outdated. Let's make sure you're using the latest software by updating the package lists:

`apt update`

Once completed, let’s update all of the currently installed packages.

`apt dist-upgrade`

It's recommended to use `apt dist-upgrade` instead of `apt upgrade` because it better handles dependencies.

Once the upgrades are complete, you'll see a list of installed packages and those no longer needed by the system.

You can remove the unnecessary packages with the following command:

`apt autoremove`

Once done reboot the server

`reboot now`

## Automatic Security Updates

Keeping your server software up to date is crucial for patching security vulnerabilities. Fortunately, Ubuntu can automatically handle software updates, ensuring your server remains secure.

*Note:* It's important to test non-security updates on a staging server before applying them, as they may introduce breaking changes that could disrupt your WordPress sites.

On some systems, automatic updates may be enabled by default. If not, or if you're unsure, follow the steps below:

Install the unattended-upgrades package:

`apt install unattended-upgrades`

Create the required configuration files:

`dpkg-reconfigure unattended-upgrades`

You should see the following screen:

![dpkg-reconfigure](https://github.com/vmzilla/PHP-Nginx-MySQL-Setup/blob/main/dpkg-reconfigure.png)

Choose “Yes” and hit Enter. Then, edit the configuration file:

`vim /etc/apt/apt.conf.d/50unattended-upgrades`

Ensure that the security origin is allowed and that all others are removed or commented out. It should look like this:

```
// Automatically upgrade packages from these (origin:archive) pairs
//
// Note that in Ubuntu security updates may pull in new dependencies
// from non-security sources (e.g. chromium). By allowing the release
// pocket these get automatically pulled in.
Unattended-Upgrade::Allowed-Origins {
            "${distro_id}:${distro_codename}";
            "${distro_id}:${distro_codename}-security";
            // Extended Security Maintenance; doesn't necessarily exist for
            // every release and this system may not have it installed, but if
            // available, the policy for updates is such that unattended-upgrades
            // should also install from here by default.
            "${distro_id}ESMApps:${distro_codename}-apps-security";
            "${distro_id}ESM:${distro_codename}-infra-security";
//          "${distro_id}:${distro_codename}-updates";
//          "${distro_id}:${distro_codename}-proposed";
//          "${distro_id}:${distro_codename}-backports";
};
```

Save and exit the file. 

You may also want to configure whether the system should automatically restart when an update requires it. By default, the server restarts immediately after an update is installed. To disable this behavior, locate the following line and uncomment it:

`Unattended-Upgrade::Automatic-Reboot "false";`

Alternatively, you can replace `false` with a specific time if you want the server to restart automatically at a scheduled time:

`Unattended-Upgrade::Automatic-Reboot-Time "04:00";`

If your server restarts, make sure to manually start any critical services that don’t restart automatically. By default, Nginx, PHP, and MySQL will restart, but you can refer to this [Stack Overflow thread](https://stackoverflow.com/questions/26267032/ubuntu-14-04-etc-init-d-vs-etc-init-start-service-at-startup) for instructions on adding other services if necessary.

Finally, set how often the automatic updates should run:

`vim /etc/apt/apt.conf.d/20auto-upgrades`

Ensure that `Unattended-Upgrade` is in the list.

`APT::Periodic::Unattended-Upgrade "1";`

Once you’ve finished editing, save and exit the file, then restart the service to have the changes take effect:

`service unattended-upgrades restart`

Now that auto security upgrades has been setup we can proceed to create our SSH user.

## Adding new user and setting up SSH

* Adding new user

We’ve completed the basic configuration of the web server and set up security updates. The next step is to create a new user on your server for two key reasons:

*We’ll be disabling SSH access for the root user later in this chapter, so you’ll need an alternative user account to access your server.
*The root user has extensive privileges that can execute potentially dangerous commands, so it’s recommended to create a new user with more restricted permissions for everyday tasks.

This new user will be added to the sudo group so that you can execute commands which require heightened permissions, but only when required.

`useradd -m -s /bin/bash -g example -d /home/example example`

Here’s a breakdown of the command:

-m : Creates the home directory for the user (/home/example).   
-s /bin/bash : Assigns the default shell (bash) to the user.   
-g example : Adds the user to the example group.   
-d /home/example : Specifies the home directory for the user.    

After running the command, you'll want to set a password for the user:

`passwd example`

This will prompt you to enter and confirm the password for the `example` user.

Next, you need to add the new user to the sudo group:

`usermod -a -G sudo example`

You can then log out and login using the new user and make sure it has shell access.

You can consider setting password less authentication for the `example` user by following the steps in the [thread here](https://www.tecmint.com/ssh-passwordless-login-using-ssh-keygen-in-5-easy-steps/)

* SSH configuration 

Now that the new user is created, it's time to enhance the server's security by configuring SSH. The first step is to disable SSH access for the root user, preventing SSH login using the root account. Open the SSH configuration file with vim:

`vim /etc/ssh/sshd_config`

The find `PermitRootLogin` and `PasswordAuthentication` and if there are already commented our make sure to uncomment them and set the values to `no`

Restart the SSH services so that the changes gets updated.

`service ssh restart`

## Configure Firewall to allow trafiic only for SSH http and https

The firewall adds an extra layer of security to your server by blocking unwanted inbound network traffic. In this guide, I’ll demonstrate how to use the `iptables` firewall, which is the most widely used on Linux systems and is typically installed by default. To make managing firewall rules easier, we use a package called `ufw` (Uncomplicated Firewall). While `ufw` is often pre-installed, if it's not, you can install it using the following command:

`sudo apt install ufw`

Now, we can start adding to the default rules, which block all incoming traffic while allowing all outgoing traffic. For now, let’s allow traffic on the ports for SSH (22), HTTP (80), and HTTPS (443):

```
sudo ufw allow ssh
sudo ufw allow http
sudo ufw allow https
```

To review which rules will be added to the firewall, enter the following command:

`sudo ufw show added`

Output:

```
Added user rules (see 'ufw status' for running firewall):
ufw allow 22/tcp
ufw allow 80/tcp
ufw allow 443/tcp
```

The run below commands to enable ufw and check the status:

```
sudo ufw enable
sudo ufw status verbose
```

## Install Fail2ban Service

Fail2ban is a security tool that complements your firewall by monitoring and blocking suspicious intrusion attempts. When it detects malicious activity, it temporarily blocks the offending IP addresses by adding them to your firewall rules. Installing Fail2ban is strongly recommended for servers running WordPress, especially if you're using third-party plugins, as it adds an extra layer of protection to secure your web server.

The Fail2ban program isn’t installed by default, so let’s install it now:

`sudo apt install fail2ban`

The default configuration should suffice, which will ban a host for 10 minutes after 6 unsuccessful login attempts via SSH. You can check default configuration in file /etc/fail2ban/jail.conf. To ensure the fail2ban service is running enter the following command:

`sudo service fail2ban start`

All set! You now have a solid foundation to start building your web server and have implemented key steps to prevent unauthorized access.


# Install Nginx, PHP 8.4, and MySQL

Now, we will begin the process of setting up Nginx, PHP-FPM, and MySQL — collectively known as the LEMP stack on Linux — to build the foundation of a functional PHP web application and server.

## Install Nginx

Nginx has become the most popular web server software for Linux servers, making it a logical choice over Apache. While the official Ubuntu package repository does include Nginx, the versions there are often outdated. To ensure you get the latest stable version, we’ll use the package repository maintained by Ondřej Surý.

Start by adding the repository and updating the package lists:

```
sudo add-apt-repository ppa:ondrej/nginx -y
sudo apt update
```

There may now be some packages that can be upgraded, let’s do that now:

`sudo apt dist-upgrade -y`

Then install Nginx:

`sudo apt install nginx -y`

Once complete, you can confirm that Nginx has been installed with the following command:

`sudo nginx -v`

Output:

```
example@vmzilla:~$ nginx -v
nginx version: nginx/1.26.1
```

Now, try visiting the domain name pointing to your server’s IP address in your browser. You should see the Nginx welcome page. Be sure to type in http://, as browsers now default to https://, which won't work yet since SSL hasn't been set up.

![nginx-default](https://github.com/vmzilla/PHP-Nginx-MySQL-Setup/blob/56e86161909c288532fd6801b3bd79a2bb89b384/nginx-default.png)

Now that Nginx is successfully installed, it's time to make some basic configurations. While Nginx comes optimized out of the box, there are a few adjustments to improve performance. Before editing the configuration file, though, you’ll need to check your server’s CPU core count and the open file limit.

Enter the following command to get the number of CPU cores your server has available. Take note of the number as we’ll use it in a minute:

`sudo grep processor /proc/cpuinfo | wc -l`

Run the following to get your server’s open file limit and take note, we’ll need it as well:

`sudo ulimit -n`

Next, open the Nginx configuration file, which can be found at /etc/nginx/nginx.conf which would look something like this:

```
user www-data;
worker_processes auto;
pid /run/nginx.pid;
error_log /var/log/nginx/error.log;
include /etc/nginx/modules-enabled/*.conf;
events {
        worker_connections 768;
        # multi_accept on;
}

http {

        ##
        # Basic Settings
        ##
```

Begin by setting the user to the username you're currently logged in with. In my case its `example`. This will simplify managing file permissions in the future. However, this is only secure if the server is accessed by a single user.

The `worker_processes` directive specifies how many worker processes to spawn per server. A good rule of thumb is to set this to match the number of CPU cores available on your server. In my case, that's 2.

The `events` block includes two key directives. First, the `worker_connections` directive should be set to your server's open file limit. This defines how many simultaneous connections each `worker_process` can handle. For example, if you have two CPU cores and an open file limit of 1024, your server can handle 2048 connections. However, the number of connections doesn't directly correspond to the number of users, as most web pages and browsers establish at least two connections per request.

Additionally, the `multi_accept` directive should be uncommented and set to `on`. This tells each `worker_process` to accept all new connections at once, rather than accepting one connection at a time.

Further down the file, you'll find the `http` block. The first directive to add is `keepalive_timeout`. This directive determines how long a connection to the client should remain open before Nginx closes it. It's a good idea to lower this value, as you don’t want idle connections lingering for up to 75 seconds when they could be used by new clients. I have set mine to 15 seconds. You can add this directive just above the `sendfile on;` directive.

```
http {

        ##
        # Basic Settings
        ##

        keepalive_timeout 15;
        sendfile on;
```

For security purposes, uncomment the `server_tokens` directive and set it to `off`. This will prevent Nginx from displaying its version number in error messages and response headers.

Underneath `server_tokens` add the following line to set the maximum upload size you require in your application.

`client_max_body_size 64m;`

I selected a value of `64m`, but you can increase it if you encounter issues uploading large files.

Further down in the `http` block, you'll find a section for gzip compression. By default, gzip is enabled, but it's a good idea to adjust these values for better handling of static files. First, uncomment the `gzip_proxied` directive and set it to `any` to ensure all proxied request responses are gzipped. Next, uncomment the `gzip_comp_level` directive and set it to a value of `5`. This controls the compression level of responses, with a range of 1 to 9. Be cautious not to set it too high, as it can increase CPU usage. Lastly, uncomment the `gzip_types` directive, leaving the default values intact. This will ensure that JavaScript, CSS, and other file types are gzipped, along with HTML, which is always compressed by the gzip module.

You must restart Nginx for the changes to take effect. Before doing so, ensure that the configuration files contain no errors by issuing the following command:

`sudo nginx -t`

If everything looks OK, go ahead and restart Nginx:

`sudo service nginx restart`

If it’s not already running, you can start Nginx with:

`sudo service nginx start`

## Installing PHP 8.4

This section covers the installation of PHP 8.4 along with a comprehensive set of extensions that are essential for various tasks, including web development, database interaction, image manipulation, email handling, and more. The following command will install PHP 8.4 and a wide range of useful extensions to enhance its functionality.

Similar to Nginx, the official Ubuntu package repository includes PHP packages, but they may not be the latest versions. To get the most up-to-date version, I recommend using the repository maintained by Ondřej Surý. Simply add the repository and update the package lists, just like you did for Nginx.

```
sudo add-apt-repository ppa:ondrej/php -y
sudo apt update
```

Then install PHP 8.4, as well as all the PHP packages you will require. Depending on the PHP extensions your application require you can modify below:

```
sudo apt install php8.4-fpm php8.4-common php8.4-mysql \
php8.4-xml php8.4-intl php8.4-curl php8.4-gd \
php8.4-imagick php8.4-cli php8.4-dev php8.4-imap \
php8.4-mbstring php8.4-opcache php8.4-redis \
php8.4-soap php8.4-zip -y
```

You’ll see `php-fpm` in the list of packages being installed. FastCGI Process Manager (FPM) is an alternative PHP FastCGI implementation with extra features, and it works particularly well with Nginx. It’s the recommended process manager when using PHP with Nginx.

Once the installation is complete, test PHP to ensure it has been installed correctly:

`sudo php-fpm8.4 -v`

* Configure PHP 8.4 and PHP-FPM

After installing Nginx and PHP, you need to configure the user and group under which the service will run. This setup does not provide security isolation between sites by configuring separate PHP pools. Instead, we will run a single PHP pool under your user account.

Open the default pool configuration file:

`sudo vim /etc/php/8.4/fpm/pool.d/www.conf`

Change the following lines, replacing `www-data` with username we created earlier:

```
user = example
group = example

listen.owner = example
listen.group = example
```

Next, you'll need to modify your `php.ini` file to increase the maximum upload size of your application. To apply the new upload limit, you must update both this setting and the `client_max_body_size` directive in Nginx. Begin by opening your `php.ini` file:

`sudo vim /etc/php/8.4/fpm/php.ini`

Change the following lines to match the value you assigned to the `client_max_body_size` directive when configuring Nginx:

```
upload_max_filesize = 64M
post_max_size = 64M
```

While editing the `php.ini` file, let's also enable the OPcache file override setting. With this setting enabled, OPcache will serve the cached version of PHP files without checking for modifications on the file system, leading to improved PHP performance.

`opcache.enable_file_override = 1`

Save the file. Before restarting PHP, check that the configuration file syntax is correct:

`sudo php-fpm8.4 -t`

If the configuration test was successful, restart PHP using the following command:

`sudo service php8.4-fpm restart`

Now that Nginx and PHP are installed, let's verify they are working together properly. To do this, enable PHP in the default Nginx site configuration and create a PHP info file to view in your browser. While this step is optional, it’s useful to ensure that PHP files are processed correctly by the Nginx web server.

Start by uncommenting a section in the default Nginx site configuration that was created when Nginx was installed:

`sudo vim /etc/nginx/sites-available/default`

Find the section which controls the PHP scripts.

```
# pass PHP scripts to FastCGI server
#
#location ~ \.php$ {
#       include snippets/fastcgi-php.conf;
#
#       # With php-fpm (or other unix sockets):
#       fastcgi_pass unix:/run/php/php8.4-fpm.sock;
#       # With php-cgi (or other tcp sockets):
#       fastcgi_pass 127.0.0.1:9000;
#}
```

As we’re using php-fpm, we can change that section to look like this:

```
# pass PHP scripts to FastCGI server

location ~ \.php$ {
       include snippets/fastcgi-php.conf;

       # With php-fpm (or other unix sockets):
       fastcgi_pass unix:/run/php/php8.4-fpm.sock;
}
```

Save the file, and then, as before, test to ensure the configuration file was edited correctly by running below command:

`sudo nginx -t`

If everything looks okay, go ahead and restart Nginx:

`sudo service nginx restart`

Next, create an `info.php` file in the default web root, which is `/var/www/html`.

```
cd /var/www/html
sudo vim info.php
```

Add the following PHP code to that `info.php` file, and save it.

`<?php phpinfo();`

Lastly, because you set the `user` directive in your `nginx.conf` file to the user you’re currently logged in with, give that user permissions on the `info.php` file.

`sudo chown example info.php`

Now, if you navigate to the `info.php` file in your browser using the domain name you configured in Chapter 1, you should see the PHP info screen. This confirms that Nginx is able to process PHP files correctly.

![php-info](https://github.com/vmzilla/PHP-Nginx-MySQL-LEMP-Stack-Setup/blob/5743828ec1967787579c0bb5afa53bbe4409ab66/info-php.png)

Once you’ve tested this, you can go ahead and delete the `info.php` file.

