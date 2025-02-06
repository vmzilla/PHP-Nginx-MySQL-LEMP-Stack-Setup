# PHP-Nginx-MySQL-Setup

A step-by-step guide to setting up a server for hosting PHP applications with Nginx and MySQL. Includes configuration instructions for Nginx, PHP-FPM, and MySQL, along with optimisation tips.

---

# Set Up and Secure a VPS 

I prefer using Ubuntu 24.04 LTS for setting up and securing services due to its stability and long-term support. As an LTS release, it receives security updates and critical patches for five years, ensuring a reliable environment for production servers. While other distributions may offer cutting-edge features, Ubuntu LTS releases focus on providing stable and well-tested software, minimizing the risk of bugs and conflicts, making it an ideal choice for hosting PHP applications with Nginx and MySQL.

If I am settings up the server for a client first thing I like to do is set a host name!

```hostnamectl hostname host.example.com

Then point the user sub domain to server IP so I can SSH user service using `ssh root@host.example.com`

## Installing Software Updates

Even though you've just provisioned your new server, some software packages may already be outdated. Let's make sure you're using the latest software by updating the package lists:

```apt update

Once completed, let’s update all of the currently installed packages.

```apt dist-upgrade

It's recommended to use `apt dist-upgrade` instead of `apt upgrade` because it better handles dependencies.

Once the upgrades are complete, you'll see a list of installed packages and those no longer needed by the system.

You can remove the unnecessary packages with the following command:

```apt autoremove

Once done reboot the server

```reboot now


