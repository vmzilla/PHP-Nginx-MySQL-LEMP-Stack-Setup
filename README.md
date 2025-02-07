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

![dpkg-reconfigure]{dpkg-reconfigure.png}

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

If your server restarts, make sure to manually start any critical services that don’t restart automatically. By default, Nginx, PHP, and MySQL will restart, but you can refer to this [Stack Overflow thread]{https://stackoverflow.com/questions/26267032/ubuntu-14-04-etc-init-d-vs-etc-init-start-service-at-startup} for instructions on adding other services if necessary.

Finally, set how often the automatic updates should run:

`vim /etc/apt/apt.conf.d/20auto-upgrades`

Ensure that `Unattended-Upgrade` is in the list.

`APT::Periodic::Unattended-Upgrade "1";`

Once you’ve finished editing, save and exit the file, then restart the service to have the changes take effect:

`service unattended-upgrades restart`

