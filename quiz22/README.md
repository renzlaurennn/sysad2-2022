# Step 1 — Installing Apache and Updating the Firewall
The Apache web server is among the most popular web servers in the world. It’s well documented, has an active community of users, and has been in wide use for much of the history of the web, which makes it a great choice for hosting a website.

Start by updating the package manager cache. If this is the first time you’re using `sudo` within this session, you’ll be prompted to provide your user’s password to confirm you have the right privileges to manage system packages with `apt`.

`sudo apt update`

Then, install Apache with: 

`sudo apt install apache2`

You’ll also be prompted to confirm Apache’s installation by pressing Y, then ENTER.

Once the installation is finished, you’ll need to adjust your firewall settings to allow HTTP traffic. UFW has different application profiles that you can leverage for accomplishing that. To list all currently available UFW application profiles, you can run:

```sudo ufw app list```

You’ll see output like this:

```` Output
Available applications:
  Apache
  Apache Full
  Apache Secure
  OpenSSH 
  ````
Here’s what each of these profiles mean:

**Apache**: This profile opens only port `80` (normal, unencrypted web traffic).
**Apache Full**: This profile opens both port `80` (normal, unencrypted web traffic) and port 443 (TLS/SSL encrypted traffic).
**Apache Secure**: This profile opens only port `443` (TLS/SSL encrypted traffic).
For now, it’s best to allow only connections on port `80`, since this is a fresh Apache installation and you still don’t have a TLS/SSL certificate configured to allow for HTTPS traffic on your server.

To only allow traffic on port `80`, use the Apache profile:

`sudo ufw allow in "Apache"`

You can verify the change with:

`sudo ufw status`

```` Output
Status: active

To                         Action      From
--                         ------      ----
OpenSSH                    ALLOW       Anywhere                                
Apache                     ALLOW       Anywhere                  
OpenSSH (v6)               ALLOW       Anywhere (v6)                    
Apache (v6)                ALLOW       Anywhere (v6)
````

Traffic on port `80` is now allowed through the firewall.

You can do a spot check right away to verify that everything went as planned by visiting your server’s public IP address in your web browser (see the note under the next heading to find out what your public IP address is if you do not have this information already):

`http://your_server_ip`

You’ll see the default Ubuntu 20.04 Apache web page, which is there for informational and testing purposes. It should look something like this:

![alt text](https://assets.digitalocean.com/articles/how-to-install-lamp-ubuntu-18/small_apache_default_1804.png)

If you see this page, then your web server is now correctly installed and accessible through your firewall.


### How To Find your Server’s Public IP Address ###
If you do not know what your server’s public IP address is, there are a number of ways you can find it. Usually, this is the address you use to connect to your server through SSH.

There are a few different ways to do this from the command line. First, you could use the `iproute2` tools to get your IP address by typing this:

```ip addr show eth0 | grep inet | awk '{ print $2; }' | sed 's/\/.*$//'```

This will give you two or three lines back. They are all correct addresses, but your computer may only be able to use one of them, so feel free to try each one.

An alternative method is to use the `curl` utility to contact an outside party to tell you how it sees your server. This is done by asking a specific server what your IP address is:

```curl http://icanhazip.com```

Regardless of the method you use to get your IP address, type it into your web browser’s address bar to view the default Apache page.

# Step 2 — Installing MySQL # 
Now that you have a web server up and running, you need to install the database system to be able to store and manage data for your site. MySQL is a popular database management system used within PHP environments.

Again, use `apt` to acquire and install this software:

```sudo apt install mysql-server```

When prompted, confirm installation by typing `Y`, and then `ENTER`.

When the installation is finished, it’s recommended that you run a security script that comes pre-installed with MySQL. This script will remove some insecure default settings and lock down access to your database system. Start the interactive script by running:

```sudo mysql_secure_installation```

This will ask if you want to configure the VALIDATE PASSWORD PLUGIN.

``` Note: Enabling this feature is something of a judgment call. If enabled, passwords which don’t match the specified criteria will be rejected by MySQL with an error. It is safe to leave validation disabled, but you should always use strong, unique passwords for database credentials.```

Answer Y for yes, or anything else to continue without enabling.

```VALIDATE PASSWORD PLUGIN can be used to test passwords
and improve security. It checks the strength of password
and allows the users to set only those passwords which are
secure enough. Would you like to setup VALIDATE PASSWORD plugin?

Press y|Y for Yes, any other key for No:
```

If you answer “yes”, you’ll be asked to select a level of password validation. Keep in mind that if you enter `2` for the strongest level, you will receive errors when attempting to set any password which does not contain numbers, upper and lowercase letters, and special characters, or which is based on common dictionary words.

```There are three levels of password validation policy:

LOW    Length >= 8
MEDIUM Length >= 8, numeric, mixed case, and special characters
STRONG Length >= 8, numeric, mixed case, special characters and dictionary              

Please enter 0 = LOW, 1 = MEDIUM and 2 = STRONG: 1
```
Regardless of whether you chose to set up the `VALIDATE PASSWORD PLUGIN`, your server will next ask you to select and confirm a password for the MySQL **root** user. This is not to be confused with the **system root**. The database root user is an administrative user with full privileges over the database system. Even though the default authentication method for the MySQL root user dispenses the use of a password, **even when one is set**, you should define a strong password here as an additional safety measure. We’ll talk about this in a moment.

If you enabled password validation, you’ll be shown the password strength for the root password you just entered and your server will ask if you want to continue with that password. If you are happy with your current password, enter `Y` for “yes” at the prompt:

````Estimated strength of the password: 100 
Do you wish to continue with the password provided?(Press y|Y for Yes, any other key for No) : y
````
For the rest of the questions, press Y and hit the `ENTER` key at each prompt. This will remove some anonymous users and the test database, disable remote root logins, and load these new rules so that MySQL immediately respects the changes you have made.

When you’re finished, test if you’re able to log in to the MySQL console by typing:

`sudo mysql`

This will connect to the MySQL server as the administrative database user root, which is inferred by the use of sudo when running this command. You should see output like this:

```Output
Welcome to the MySQL monitor.  Commands end with ; or \g.
Your MySQL connection id is 22
Server version: 8.0.19-0ubuntu5 (Ubuntu)

Copyright (c) 2000, 2020, Oracle and/or its affiliates. All rights reserved.

Oracle is a registered trademark of Oracle Corporation and/or its
affiliates. Other names may be trademarks of their respective
owners.

Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.

mysql> 
```
To exit the MySQL console, type:

`mysql> exit`

Notice that you didn’t need to provide a password to connect as the **root** user, even though you have defined one when running the `mysql_secure_installation script`. That is because the default authentication method for the administrative MySQL user is `unix_socket` instead of `password`. Even though this might look like a security concern at first, it makes the database server more secure because the only users allowed to log in as the **root** MySQL user are the system users with sudo privileges connecting from the console or through an application running with the same privileges. In practical terms, that means you won’t be able to use the administrative database root user to connect from your PHP application. Setting a `password` for the root MySQL account works as a safeguard, in case the default authentication method is changed from `unix_socket` to password.

For increased security, it’s best to have dedicated user accounts with less expansive privileges set up for every database, especially if you plan on having multiple databases hosted on your server.

```
Note: At the time of this writing, the native MySQL PHP library mysqlnd doesn’t support caching_sha2_authentication, the default authentication method for MySQL 8. For that reason, when creating database users for PHP applications on MySQL 8, you’ll need to make sure they’re configured to use mysql_native_password instead. We’ll demonstrate how to do that in Step 6.
```

Your MySQL server is now installed and secured. Next, we’ll install PHP, the final component in the LAMP stack.

# Step 3 — Installing PHP#
You have Apache installed to serve your content and MySQL installed to store and manage your data. PHP is the component of our setup that will process code to display dynamic content to the final user. In addition to the `php` package, you’ll need `php-mysql`, a PHP module that allows PHP to communicate with MySQL-based databases. You’ll also need `libapache2-mod-php` to enable Apache to handle PHP files. Core PHP packages will automatically be installed as dependencies.

To install these packages, run:

```sudo apt install php libapache2-mod-php php-mysql```
Once the installation is finished, you can run the following command to confirm your PHP version:

```php -v```

````Output
PHP 7.4.3 (cli) (built: Jul  5 2021 15:13:35) ( NTS )
Copyright (c) The PHP Group
Zend Engine v3.4.0, Copyright (c) Zend Technologies
    with Zend OPcache v7.4.3, Copyright (c), by Zend Technologies
At this point, your LAMP stack is fully operational, but before you can test your setup with a PHP script, it’s best to set up a proper Apache Virtual Host to hold your website’s files and folders. We’ll do that in the next step.
````
At this point, your LAMP stack is fully operational, but before you can test your setup with a PHP script, it’s best to set up a proper Apache Virtual Host to hold your website’s files and folders. We’ll do that in the next step.

