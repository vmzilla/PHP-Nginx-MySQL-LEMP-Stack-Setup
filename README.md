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
