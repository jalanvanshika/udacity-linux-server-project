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
  ssh-copy-id -i ~/.ssh/udacitylinux grader@165.22.178.141
  ssh grader@165.22.178.141
  exit
  ```

- At this point, login will be accomplished by matching the *private* key that pairs with the public key you uploaded.



## Secure server

- I did this as user `grader` with `sudo`.

  ```sh
  sudo ufw app list
  sudo ufw allow OpenSSH
  sudo ufw allow 2200
  sudo ufw allow 2200/tcp
  sudo ufw allow 80/tcp
  sudo ufw allow 123/udp
  sudo ufw enable
  ```

  ```text
  $ sudo ufw status
  Status: active

  To                         Action      From
  --                         ------      ----
  OpenSSH                    ALLOW       Anywhere
  2200/tcp                   ALLOW       Anywhere
  80/tcp                     ALLOW       Anywhere
  123/udp                    ALLOW       Anywhere
  2200                       ALLOW       Anywhere
  OpenSSH (v6)               ALLOW       Anywhere (v6)
  2200/tcp (v6)              ALLOW       Anywhere (v6)
  80/tcp (v6)                ALLOW       Anywhere (v6)
  123/udp (v6)               ALLOW       Anywhere (v6)
  2200 (v6)                  ALLOW       Anywhere (v6)
  ```

- Change SSH port from 22 to 2200
  - We were required to do this for the Udacity project.
  - Open the configuration file and edit the file with the nano text editor.

    ```sh
    sudo nano /etc/ssh/sshd_config
    ```

    ```text
    # What ports, IPs and protocols we listen for
    Port 2200
    ```

  - **Important**: Disable password authentication in `sshd_config` so SSH is required for login, and disable root login so user must specify `sudo`:

    ```text
    # Change to no to disable tunnelled clear text passwords
    PasswordAuthentication no
    PermitRootLogin no
    ```

  - Save and quit with ctrl+x.
  - Restart SSH

    ```sh
    sudo systemctl reload sshd
    sudo service ssh restart
    ```

  - Exit and log back in, this time specifying port 2200.

    ```sh
    logout
    ssh grader@165.22.178.141 -p 2200
    ```
- The *~/.ssh/config* file on your local machine can also be configured for easier login. This file is on my local machine, so I was able to to open it with vscode.

  ```sh
  $ code ~/.ssh/config
  ```

  ```text
  Host udacitylinux
    Hostname 165.22.178.141
    User grader
    Port 2200
    PubKeyAuthentication yes
    IdentityFile ~/.ssh/udacitylinux

  Host *
    AddKeysToAgent yes
    UseKeychain yes
    IdentityFile ~/.ssh/id_rsa
  ```

- If the config file is set up as above, log in with `ssh udacitylinux`.

- **After disabling password login, I started getting the SSH error `Permission denied (publickey).`** I couldn't log in from my local terminal. I spent hours troubleshooting it and working in the DigitalOcean browser console.  I read resources including [SSH.com](https://www.ssh.com/iam/ssh-key-management/) and [askubuntu](https://askubuntu.com/questions/311558/ssh-permission-denied-publickey), but the solutions didn't help. Eventually, I realized that the best solution was:
  - Delete the SSH key
  - Re-do the keygen as described above.
  - Temporarily enable password login again from the DigitalOcean browser console.
  - Add the new key with `ssh-copy-id` as described above, and authenticate with user password.
  - Log in with SSH.
  - Disable password authentication.
  - Exit
  - Log back in to verify SSH.

[(Back to top)](#top)









ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABAQDbE91jtYqauFIGdagYB5PKZMdvG12CXcD4c3DcVrVdi6XAy1nYhjD3zwhFN+j1D9jLmru5X7Ro4dDKSJPnTCMQpW7Znen7J6cmW3pP92DEMZwo7ISwCk8XZ8GTtPCvV6VtapIbe04rrTbIxexEt0kUeOwU0MYsCa74TxGxXd07zfn2t7IbV4pdxF2ddj1Erf6BxDXiR0TKJaR9VOYJ8rwN+k9JmtpS9pHCQipme0roHs6fZIt/PCE/mgkzJJWH+Ag6CE5I00xSwsvvQjW0LD43NDvDFES7SEt/TpJVqc/iRapqNjR/wooPvyHt+XvSLI10IkpFKiP51JRzLo3hjXUz vagrant@vagrant



