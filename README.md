## Table of Contents <!-- omit in toc -->

- [Select a server host](#select-a-server-host)
- [Set up server](#set-up-server)
- [Set up user](#set-up-user)
- [Set up SSH](#set-up-ssh)
- [Secure server](#secure-server)
- [Deploy app](#deploy-app)
  - [Configure PostgreSQL](#configure-postgresql)
  - [Clone app files](#clone-app-files)
  - [Set up Python environment](#set-up-python-environment)
  - [Configure web server](#configure-web-server)
- [Domain name](#domain-name)
- [SSL](#ssl)
- [Database maintenance](#database-maintenance)
- [Docker deployment](#docker-deployment)
- [Troubleshooting](#troubleshooting)

## Select a server host
I am using digital ocean

## Set up server

- It was easy to set up my DigitalOcean droplet. I just followed the on-screen instructions, but there is also a [tutorial](info@news.digitalocean.com) available if needed.
- I chose a $5 base-level droplet.
- I did not set up the SSH key during droplet creation. See below for SSH setup.
- Connect to the server
- Update and upgrade packages, then reboot

  ```sh
  sudo apt update
  sudo apt upgrade
  sudo reboot
  ```

- Set time zone to UTC:

  ```sh
  sudo dpkg-reconfigure tzdata
  ```

  - Select "None of these," then find UTC in the list and press enter.
- If you ever need to restart the server, run either `sudo reboot`, or `sudo shutdown -h now` and turn the server back on through the website interface.

[(Back to top)](#top)

## Set up user

- I continued by following the [DigitalOcean Initial Server Setup with Ubuntu 16.04 tutorial](https://www.digitalocean.com/community/tutorials/initial-server-setup-with-ubuntu-16-04).
- The first step is logging in as root:

  ```sh
  ssh root@165.22.178.141
  ```

- At this point, root can log in with the password sent to your email address by DigitalOcean, and you can then change the password after login.
- After logging in as root, I created two users: `br3ndonland` for me and `grader` for the Udacity grader. I left the `grader` password `grader`. I gave each user `sudo` privileges.

  ```sh
  adduser jalanvanshika
  adduser grader
  usermod -aG sudo jalanvanshika
  usermod -aG sudo grader
  ```

- Users can be deleted later with `[userdel](http://manpages.ubuntu.com/manpages/trusty/man8/userdel.8.html)` or `deluser`: `userdel grader`.
- View list of users with `cut -d: -f1 /etc/passwd`

## Set up SSH

- See the [DigitalOcean How To Connect To Your Droplet with SSH](https://www.digitalocean.com/community/tutorials/how-to-connect-to-your-droplet-with-ssh) guide.
- I [generated an SSH key and added it to the SSH agent](https://help.github.com/articles/generating-a-new-ssh-key-and-adding-it-to-the-ssh-agent/) *on my local machine,* with the method recommended by GitHub, which attaches your email instead of the local machine name. The config file may need to be manually created with `touch ~/.ssh/config` first. I named the key `udacity6`.

  ```sh
  $ touch ~/.ssh/config
  $ ssh-keygen -t rsa -b 4096 -C "your_email@example.com"
    # Enter file in which to save the key (/home/vagrant/.ssh/id_rsa):/home/vagrant/.ssh/udacitylinux
    # Your identification has been saved in /home/vagrant/.ssh/udacitylinux.
    # Your public key has been saved in /Users/br3ndonland/.ssh/udacitylinux.pub.
  $ eval "$(ssh-agent -s)"
  $ ssh-add -K ~/.ssh/udacitylinux
  ```

- I copied the SSH key to the server for each user with `ssh-copy-id`. **Note that `ssh-copy-id` relies on password authentication, so if you disable password authentication this won't work.** Copy the SSH ID and verify login before disabling password authentication. If you have already reconfigured your ssh port to 2200, add `-p 2200`. Also, be sure to reference the *private* key when using `ssh-copy-id`, and not the public key (in this example, *udacity6* instead of *udacity6.pub*).

  ```sh
  ssh-copy-id -i ~/.ssh/udacitylinux jalanvanshika@165.22.178.141
  ssh jalanvanshika@165.22.178.141
  exit
  ssh-copy-id -i ~/.ssh/udacity6 grader@165.22.178.141
  ssh grader@165.22.178.141
  exit
  ```

- At this point, login will be accomplished by matching the *private* key that pairs with the public key you uploaded.












ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABAQDbE91jtYqauFIGdagYB5PKZMdvG12CXcD4c3DcVrVdi6XAy1nYhjD3zwhFN+j1D9jLmru5X7Ro4dDKSJPnTCMQpW7Znen7J6cmW3pP92DEMZwo7ISwCk8XZ8GTtPCvV6VtapIbe04rrTbIxexEt0kUeOwU0MYsCa74TxGxXd07zfn2t7IbV4pdxF2ddj1Erf6BxDXiR0TKJaR9VOYJ8rwN+k9JmtpS9pHCQipme0roHs6fZIt/PCE/mgkzJJWH+Ag6CE5I00xSwsvvQjW0LD43NDvDFES7SEt/TpJVqc/iRapqNjR/wooPvyHt+XvSLI10IkpFKiP51JRzLo3hjXUz vagrant@vagrant



