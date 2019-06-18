# Linux Server Configuration

## Contents <!-- omit in toc -->

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
  ssh root@159.65.159.60
  ```

- At this point, root can log in with the password sent to your email address by DigitalOcean, and you can then change the password after login.
- After logging in as root, I created two users: `jalanvanshika` for me and `grader` for the Udacity grader. I left the `grader` password `grader`. I gave each user `sudo` privileges.

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
- I [generated an SSH key and added it to the SSH agent](https://help.github.com/articles/generating-a-new-ssh-key-and-adding-it-to-the-ssh-agent/) *on my local machine,* with the method recommended by GitHub, which attaches your email instead of the local machine name. The config file may need to be manually created with `touch ~/.ssh/config` first. I named the key `udacityserver`.

  ```sh
  $ touch ~/.ssh/config
  $ ssh-keygen -t rsa -b 4096 -C "your_email@example.com"
    # Enter file in which to save the key (/home/vagrant/.ssh/id_rsa):/home/vagrant/.ssh/udacityserver
    # Your identification has been saved in /home/vagrant/.ssh/udacityserver.
    # Your public key has been saved in /home/vagrant/.ssh/udacityserver.pub.
  $ eval "$(ssh-agent -s)"
  $ ssh-add -K ~/.ssh/udacityserver
  ```

- I copied the SSH key to the server for each user with `ssh-copy-id`. **Note that `ssh-copy-id` relies on password authentication, so if you disable password authentication this won't work.** Copy the SSH ID and verify login before disabling password authentication. If you have already reconfigured your ssh port to 2200, add `-p 2200`. Also, be sure to reference the *private* key when using `ssh-copy-id`, and not the public key (in this example, *udacity6* instead of *udacity6.pub*).

  ```sh
  ssh-copy-id -i ~/.ssh/udacitylinux jalanvanshika@159.65.159.60
  ssh jalanvanshika@159.65.159.60
  exit
  ssh-copy-id -i ~/.ssh/udacitylinux grader@159.65.159.60
  ssh grader@159.65.159.60
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
    ssh grader@159.65.159.60 -p 2200
    ```
- The *~/.ssh/config* file on your local machine can also be configured for easier login. This file is on my local machine, so I was able to to open it with vscode.

  ```sh
  $ code ~/.ssh/config
  ```

  ```text
  Host udacitylinux
    Hostname 159.65.159.60
    User grader
    Port 2200
    PubKeyAuthentication yes
    IdentityFile ~/.ssh/udacityserver

  Host *
    AddKeysToAgent yes
    UseKeychain yes
    IdentityFile ~/.ssh/id_rsa
  ```

- If the config file is set up as above, log in with `ssh udacityserver`.

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

## Deploy app

DigitalOcean has helpful documentation for the Linux server itself, but less documentation for applications. They have [one-click app support for Django](https://www.digitalocean.com/community/tutorials/how-to-use-the-django-one-click-install-image-for-ubuntu-16-04), but not for Flask. 

### Configure PostgreSQL

- My app uses SQLite, but Postgres is a more flexible choice for a multi-user application database. Create a PostgreSQL database for the app. The `psql` command activates the SQL interpreter, and the words in capital letters are SQL commands. Note the prompt changes after logging in to the database.

  ```sh
  $ sudo apt-get install libpq5
  $ sudo apt-get install postgresql
  $ sudo su - postgres
  postgres@udacity6:~$ psql
  psql (9.5.12)
  Type "help" for help.

  postgres=CREATE USER catalog WITH PASSWORD 'grader';
  CREATE ROLE
  postgres=ALTER USER catalog CREATEDB;
  ALTER ROLE
  postgres=CREATE DATABASE catalog WITH OWNER catalog;
  CREATE DATABASE
  postgres=\c catalog
  You are now connected to database "catalog" as user "postgres".
  catalog=REVOKE ALL ON SCHEMA public FROM public;
  REVOKE
  catalog=GRANT ALL ON SCHEMA public TO catalog;
  GRANT
  catalog=\q
  postgres@udacity6:~$ exit
  logout
  ```

- See below for more info on [database maintenance](#database-maintenance).
- Changing from SQLite to PostgreSQL will require installation of `psycopg2` after Python is installed.

### Clone app files

- Git should already be installed.

  ```sh
  $ which git
  /usr/bin/git
  ```

- Clone [app repo](https://github.com/jalanvanshika/udacity-item-catalog-project) and create directory in a single step:

  ```sh
  sudo git clone https://github.com/jalanvanshika/udacity-item-catalog-project /var/www/catalog
  ```

  The app is now located in `/var/www/`, the directory in which Apache allows sites, or "virtual host configurations."

  ```text
  cd /var/www/catalog
  ```

- Change owner of catalog directory:

  ```sh
  sudo chown -R grader:grader /var/www/catalog
  ```

### Set up Python environment

#### Configure app files

- Copy the client secret from your [Google Cloud APIs credentials dashboard](https://console.cloud.google.com/apis/credentials) into the *client_secrets.json* file on the server. Make sure the server's IP, and your domain name if you have one, have been added to the "Authorized JavaScript origins" and "Authorized redirect URIs."

  ```sh
  cd /var/www/catalog
  sudo touch client_secrets.json
  sudo nano client_secrets.json
  ```

- Modify the *project.py* file for the server:
  - Move `app.secret_key` out of the `if __name__ == '__main__'` block (which will not be used by the WSGI server), as recommended [here](https://stackoverflow.com/a/33898263).
  - Change paths to the *client_secrets.json* file, and any other external files, to absolute. Remember to modify the path in the Google Sign-In code.

  ```sh
  cd /var/www/catalog
  sudo nano project.py
  ```

  ```python
  # Obtain credentials from JSON file
  CLIENT_ID = json.loads(open('/var/www/catalog/client_secrets.json', 'r')
                        .read())['web']['client_id']
  CLIENT_SECRET = json.loads(open('/var/www/catalog/client_secrets.json', 'r')
                            .read())['web']['client_secret']
  redirect_uris = json.loads(open('/var/www/catalog/client_secrets.json', 'r')
                            .read())['web']['redirect_uris']
  app.secret_key = CLIENT_SECRET


- Configure *database_setup.py*, *database_data.py*, and *application.py* for PostgreSQL instead of SQLite.

  ```python
  # delete this
  # engine = create_engine('sqlite:///catalog.db')
  # and add this
  engine = create_engine('postgresql://catalog:grader@localhost/catalog')
  ```

#### Set up Python

- Install Python 3.6
  - My virtual environment requires Python 3.6, which is not available from Ubuntu apt yet. Python 3.6 must be installed as a third-party application.
  - I followed the instructions [here](https://askubuntu.com/a/865569) to install:

    ```sh
    sudo add-apt-repository ppa:deadsnakes/ppa
    sudo apt update
    sudo apt install python3.6
    ```

  - Python 3.6 must be specified with `python3.6` when python is run, or when the virtual environment is configured.

- Install pip

  ```sh
  sudo apt install python3-pip
  ```

#### Run app without virtual environment

**Note that, despite extensive troubleshooting, I was not able to configure WSGI to serve up the Python Flask app from a virtual environment. The way I got the app running was by installing required packages without a virtual environment.**

- Install required packages

  ```sh
  sudo pip3 install flask
  sudo pip3 install oauth2client
  sudo pip3 install psycopg2
  sudo pip3 install requests
  sudo pip3 install sqlalchemy
  ```

- Run application files

  ```sh
  python3.6 database_setup.py
  python3.6 database_data.py
  python3.6 application.py
  ```

- The Flask app should now be running on localhost. Stop the local server with ctrl+c and continue with the configuration process.


### Configure web server

#### WSGI

- Create WSGI script.
  - It seems like the script can either be named `catalog.wsgi` or `wsgi.py`, as long as the script name is properly specified in the Apache configuration file.
  - The script has a few parts:
    - Enabling the virtual environment: As explained above, I wasn't able to get WSGI to serve the app from the virtual environment, so I deleted this part. In the future, if configuration is successful, code added to *wsgi.py* would look something like this (based on the [Flask mod_wsgi docs](http://flask.pocoo.org/docs/1.0/deploying/mod_wsgi/#working-with-virtual-environments), modified for pipenv):

      ```python
      # Enable use of virtual environment with mod_wsgi
      activate_this = '/home/grader/.local/share/virtualenvs/catalog-1BsMKvn0/bin/activate_this.py'
      with open(activate_this) as file_:
          exec(file_.read(), dict(__file__=activate_this))
      ```

    - Specifying the path to the application directory: It seems like this path is only used within *wsgi.py* to find the specific *application.py* file. File paths used within *application.py* will still need to be changed to absolute.
    - Importing the Flask app: The syntax requirements here were a little difficult to figure out also. I originally had `from catalog import app as application`, which didn't seem to be working. It helped to open the *wsgi.py* file in my local text editor (vscode), because the linting showed me that `catalog` was not found. I needed to import the Flask instance (`app`) from the application file *application.py*, which would be `from application import app as application`.
    - Name main block: This seems to be used instead of the name/main block in the main application file. I added the filesystem session type as recommended [here](https://stackoverflow.com/a/33898263).
- Here's what a completed WSGI script should look like:

  ```sh
  sudo nano /var/www/catalog/wsgi.py
  ```

  ```python
  # Add path to application directory
  import sys
  sys.path.insert(0, "/var/www/catalog")

  # Import Flask instance from main application file
  from application import app as application

  if __name__ == '__main__':
      app.config['SESSION_TYPE'] = 'filesystem'
      app.run(host='0.0.0.0', port=8000)

  ```

#### Apache

- Install and configure Apache to serve a Python `mod_wsgi` application. The [Flask docs](http://flask.pocoo.org/docs/1.0/deploying/mod_wsgi/) have some simple instructions.
- Install Apache and Python 3 `mod_wsgi`

  ```sh
  sudo apt-get install apache2
  sudo apt-get install libapache2-mod-wsgi-py3
  ```

  - Some guides also suggest running `sudo a2enmod wsgi`, but WSGI should already enable itself upon installation.
- Browsing to the server ip [http://192.241.141.20/](http://192.241.141.20/) should show the "It works!" default Apache page.
  > Apache2 Ubuntu Default Page
  >
  > It works!
  >
  > This is the default welcome page used to test the correct operation of the Apache2 server after installation on Ubuntu systems.
- Add Apache [configuration file](http://httpd.apache.org/docs/current/configuring.html) `catalog.conf`.
  - `ServerName`: Server's IP address.
  - `ServerAdmin`: Optionally enter your email address here.
  - `WSGIDaemonProcess`: Helps the app run when logged in as a different user.
  - `WSGIScriptAlias`: There is an initial `/`, followed by the absolute file path to the WSGI script.
  - The `Directory` section grants access to the directory containing the application files.
  - The `ErrorLog` and `CustomLog` are particularly helpful for debugging.
- Here's how the completed Apache configuration file will look:

  ```sh
  sudo nano /etc/apache2/sites-available/catalog.conf
  ```

  ```text
  <VirtualHost *:80>
    ServerName 192.241.141.20
    ServerAdmin brendon.w.smith@gmail.com
    WSGIDaemonProcess application user=br3ndonland threads=3
    WSGIScriptAlias / /var/www/catalog/wsgi.py
    <Directory /var/www/catalog/>
      WSGIProcessGroup application
      WSGIApplicationGroup %{GLOBAL}
      Require all granted
    </Directory>
    ErrorLog ${APACHE_LOG_DIR}/error.log
    LogLevel warn
    CustomLog ${APACHE_LOG_DIR}/access.log combined
  </VirtualHost>
  ```

- Enable virtual host (should only need to be done once):

  ```sh
  sudo a2ensite catalog
  sudo service apache2 restart
  ```

[(Back to top)](#top)

## Domain name

- DigitalOcean is not a DNS registrar at this time. I followed the [DigitalOcean DNS tutorial](https://www.digitalocean.com/community/tutorials/an-introduction-to-digitalocean-dns) to add a domain name purchased through [Hover](https://www.hover.com/).
- From my Hover dashboard, I pointed the domain to DigitalOcean's nameservers.

  ```text
  ns1.digitalocean.com
  ns2.digitalocean.com
  ns3.digitalocean.com
  ```

- In my DigitalOcean account, from the Networking tab, I set an A record (for ipv4) and an AAAA record (for ipv6) so that the hostname catalog.br3ndonland.com directs to the server's IP.

[(Back to top)](#top)


ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABAQDbE91jtYqauFIGdagYB5PKZMdvG12CXcD4c3DcVrVdi6XAy1nYhjD3zwhFN+j1D9jLmru5X7Ro4dDKSJPnTCMQpW7Znen7J6cmW3pP92DEMZwo7ISwCk8XZ8GTtPCvV6VtapIbe04rrTbIxexEt0kUeOwU0MYsCa74TxGxXd07zfn2t7IbV4pdxF2ddj1Erf6BxDXiR0TKJaR9VOYJ8rwN+k9JmtpS9pHCQipme0roHs6fZIt/PCE/mgkzJJWH+Ag6CE5I00xSwsvvQjW0LD43NDvDFES7SEt/TpJVqc/iRapqNjR/wooPvyHt+XvSLI10IkpFKiP51JRzLo3hjXUz vagrant@vagrant



