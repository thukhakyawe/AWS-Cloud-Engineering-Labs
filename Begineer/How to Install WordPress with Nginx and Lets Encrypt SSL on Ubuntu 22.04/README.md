# How-to-Install-WordPress-with-Nginx-and-Let-s-Encrypt-SSL-on-Ubuntu-22.04

![alt text](image.png)

- To install WordPress with Nginx and Let's Encrypt SSL on Ubuntu 22.04, you'll need to follow several steps. Here's a guide to walk you through the process:

1. <h3>Update System Packages:</h3>

- First, update your Ubuntu system to ensure you have the latest packages:

```console
sudo apt update
sudo apt upgrade
```
2. <h3>Install Nginx:</h3>

- Install Nginx using the following command:

```console
sudo apt install nginx
```

3. <h3>Install MySQL:</h3>

- WordPress requires a database. Install MySQL using the following command:

```console
sudo apt install mysql-server
```

During installation, you'll be prompted to set up a root password.

4. <h3>Install PHP: </h3>
- Install PHP and required extensions:

```console
sudo apt install php-fpm php-mysql php-curl php-gd php-mbstring php-xml php-xmlrpc php-soap php-intl php-zip
```


5. <h3>Configure Nginx:</h3> 
- Create a new server block configuration for your WordPress site:

```console
sudo vim /etc/nginx/sites-available/your_domain
```

- Replace `your_domain` with `your actual domain name`. Here's a basic configuration:

```console
server {
    listen 80;
    server_name your_domain.com www.your_domain.com;
    root /var/www/your_domain;
    index index.php index.html index.htm;
    location / {
        try_files $uri $uri/ /index.php?$args;
    }
    location ~ \.php$ {
        include snippets/fastcgi-php.conf;
        fastcgi_pass unix:/var/run/php/php7.4-fpm.sock;
        fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
        include fastcgi_params;
    }
}
```
- Save and close the file.

- Enable the Nginx server block:

```console
sudo ln -s /etc/nginx/sites-available/your_domain /etc/nginx/sites-enabled/
```

- Test Nginx configuration:

```console
sudo nginx -t
```

- If the test is successful, reload Nginx:

```console
sudo systemctl reload nginx
```

6. <h3> Create MySQL Database and User: </h3>
- Log in to the MySQL console and create a database and user for WordPress:

```Console
sudo mysql -u root -p
```

```console
CREATE DATABASE wordpress;
CREATE USER 'wordpressuser'@'localhost' IDENTIFIED BY 'your_password';
GRANT ALL PRIVILEGES ON wordpress.* TO 'wordpressuser'@'localhost';
FLUSH PRIVILEGES;
EXIT;
```


7. <h3> Install Certbot and Obtain SSL Certificate: </h3>
- Install Certbot to get a free Let's Encrypt SSL certificate:

```console
sudo apt install certbot python3-certbot-nginx
```

- Then, run Certbot to obtain and install the SSL certificate:

```console
sudo certbot --nginx -d your_domain.com -d www.your_domain.com
```

- Follow the prompts to configure your SSL certificate.

8. <h3> Download and Configure WordPress: </h3>
- Download the latest WordPress:

```console
cd /tmp
wget https://wordpress.org/latest.tar.gz
tar -zxvf latest.tar.gz
```

- Move WordPress files to your Nginx web root directory:

```console
sudo mv /tmp/wordpress /var/www/your_domain
```

- Set appropriate permissions:

```console
sudo chown -R www-data:www-data /var/www/your_domain
sudo chmod -R 755 /var/www/your_domain
```

- Copy the sample configuration file:

```console
cp /var/www/your_domain/wp-config-sample.php /var/www/your_domain/wp-config.php
```

- Edit the configuration file and add your MySQL database details:

```console
sudo vim /var/www/your_domain/wp-config.php
```

- Replace the following lines with your MySQL database, username, and password:

```console
define( 'DB_NAME', 'wordpress' );
define( 'DB_USER', 'wordpressuser' );
define( 'DB_PASSWORD', 'your_password' );
```

- Save and close the file.

9. <h3> Finish Installation via Browser: </h3>

- Now, you can complete the WordPress installation by visiting ` http://your_domain.com` in your web browser and following the on-screen instructions.

**_That's it! You now have WordPress installed with Nginx and Let's Encrypt SSL on Ubuntu 22.04._**