# Server Setup Guide

This guide will walk you through the process of setting up a web server with Apache, PHP, MySQL, and Laravel on a Linux server. Follow the steps below to get your server up and running.

## Update Package List

```bash
sudo apt update
```

## Install Apache Web Server

```bash
sudo apt install apache2
```

## Allow Access to Apache

```bash
sudo ufw allow "Apache Full"
```

## Install UFW (if not installed)

```bash
sudo apt install ufw
```

## Test Apache Server

Open a web browser and enter your server's IP address to check if the Apache server is running.

## Install PHP and Necessary Extensions

```bash
sudo apt install php libapache2-mod-php php-mbstring php-gd php-xml php-cli php-zip php-json php-curl php-intl php-mysql
```

## Create a PHP Test File

```bash
sudo nano /var/www/html/test.php
```

Add the following PHP code to the file and save it:

```php
<?php
echo phpinfo();
?>
```

Save and Exit (ctr+o ctr+x) the text editor and check the PHP test file in your browser (IP Address + `/test.php`).

## Install MySQL Server

```bash
sudo apt install mysql-server
```

## Configure MySQL

```bash
sudo mysql
```

Enter the following commands in the MySQL shell:

```sql
ALTER USER 'root'@'localhost' IDENTIFIED WITH mysql_native_password BY 'SetRootPasswordHere';
exit
```

## Secure MySQL Installation

```bash
sudo mysql_secure_installation
```

Follow the prompts to secure your MySQL installation.

- validate password component (no)
- change the root password (no)
- remove anonymous users (yes)
- disable rool login remotely  (yes)
- remove test database and access to it (yes)
- reload privilege table now (yes)


## Create a Database

```bash
sudo mysql
```

In the MySQL shell, create a database for your Laravel project:

```sql
CREATE DATABASE laravelexample CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;
```

## Install Git

```bash
sudo apt install git
```
Note : set username and email and create password from git and use it while cloning

## Clone Your Git Repository

Navigate to the `/var/www/html` directory and clone your Git repository:

```bash
cd /var/www/html
git clone your_git_repo
```

## Install Composer

Visit [Composer's download page](https://getcomposer.org/download/#:~:text=Command%2Dline%20installation) and run the following commands:

```bash
php -r "copy('https://getcomposer.org/installer', 'composer-setup.php');"
php -r "if (hash_file('sha384', 'composer-setup.php') === 'e21205b207c3ff031906575712edab6f13eb0b361f2085f1f1237b7126d785e826a450292b6cfd1d64d92e6563bbde02') { echo 'Installer verified'; } else { echo 'Installer corrupt'; unlink('composer-setup.php'); } echo PHP_EOL;"
php composer-setup.php
php -r "unlink('composer-setup.php');"
sudo mv composer.phar /usr/local/bin/composer
composer
```

## Configure Laravel Environment

Navigate to your Laravel project directory and install dependencies:

```bash
composer install
cp .env.example .env
sudo nano .env
```

Edit the `.env` file to set your database connection details:

```env
DB_CONNECTION=mysql
DB_HOST=127.0.0.1
DB_PORT=3306
DB_DATABASE=laravel_example
DB_USERNAME=root
DB_PASSWORD={root_user_password}
```

## Run Database Migrations

```bash
php artisan migrate
```

## Configure File Permissions

```bash
sudo chgrp -R www-data /var/www/html/laravelexample
ls -la
sudo chmod -R 775 /var/www/html/laravelexample/storage
```

## Create Apache Configuration

```bash
cd /etc/apache2/sites-available
sudo nano laravelexample.net.conf
```

Add the following configuration to the `laravelexample.net.conf` file:

```apache
<VirtualHost *:80>
    ServerName laravelexample.net
    ServerAdmin webmaster@thedomain.com
    DocumentRoot /var/www/laravelexample/public
    <Directory /var/www/laravelexample>
        AllowOverride All
    </Directory>
    ErrorLog ${APACHE_LOG_DIR}/error.log
    CustomLog ${APACHE_LOG_DIR}/access.log combined
</VirtualHost>
```

Save and exit the text editor.

## Enable Apache Configuration

```bash
sudo a2dissite 000-default.conf
sudo a2ensite laravelexample.net.conf
sudo a2enmod rewrite
sudo service apache2 restart
```

## Set Up Domain

On your domain provider's dashboard, in the DNS and name server section, configure A and CNAME records to point to your server's IP address.

## Optional: Node.js Setup

If you need to use Node.js modules, follow these steps:

```bash
curl -fsSL https://deb.nodesource.com/setup_18.x | sudo -E bash -
sudo apt-get install -y nodejs
npm install
npm run build
```

## Optional: SSL Installation

To enable HTTPS, install Certbot and configure it:

```bash
sudo apt install certbot python3-certbot-apache
sudo certbot --apache
```

Follow the prompts to configure HTTPS for your domain.

Your server should now be set up and running. Access your Laravel project via your domain name or IP address.
```

This markdown file provides clear and organized instructions for setting up your server step by step. Feel free to use it as your README.md file.
