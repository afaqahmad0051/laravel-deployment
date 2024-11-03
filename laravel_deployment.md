# Comprehensive Laravel Deployment Guide for Ubuntu VPS

Few considerations before starting to setup the server.

- Choose Ubuntu LTS as operating system.
- Assign a flexible IP to server if possible
    - It will remain available to you even if you delete the server, ensuring consistent connectivity if you plan to switch servers, as you can reassign the same IP without changing DNS.
- Point the domain/sub-domain to server static IP, as it can take some time.
- Run `apt update` before installing packages ensures that your server's package list is up-to-date. This means when you install or upgrade software, you get the latest available versions and security patches. Without updating, you might install outdated or vulnerable packages, which could cause compatibility issues or security risks.
- It's recommended to use public key authentication over passwords for server access. Public keys are far more secure, as they rely on cryptographic pairs, making it much harder for attackers to break in. By using keys, you can also disable password-based logins entirely, reducing exposure to brute-force attacks and phishing attempts. This is a simple but effective way to improve server security.

## 1. User Setup and Security

Before we begin the deployment process, let's set up proper user accounts for enhanced security.

*This guide use **dev** name as super user if you want a different name, you will have to modify the commands.*


1. Create a superuser named 'dev':
   ```
   sudo adduser dev
   sudo usermod -aG sudo dev
   ```
   Why: Creating a superuser separate from the root account adds an extra layer of security. This user can perform administrative tasks without logging in as root.

   If you want to disable root login, you can set `PermitRootLogin no` in `/etc/ssh/sshd_config` and restart the SSH service with `sudo systemctl restart ssh`.

   If dev want to use public key for login.

   Create `.ssh` directory and set permissions:
   ```
   sudo mkdir -p /home/dev/.ssh
   sudo touch /home/dev/.ssh/authorized_keys
   sudo chown -R dev:dev /home/dev/.ssh
   sudo chmod 700 /home/dev/.ssh
   sudo chmod 600 /home/dev/.ssh/authorized_keys
   ```
   Why: These commands create the necessary `.ssh` directory and `authorized_keys` file, set the correct ownership, and apply the appropriate permissions to ensure secure SSH key-based authentication.

   To add your public key to the `authorized_keys` file, use the following command:
   ```
   echo "your-public-key" | sudo tee -a /home/dev/.ssh/authorized_keys
   ```
   Replace `your-public-key` with your actual SSH public key.

   Why: This appends your public key to the `authorized_keys` file, allowing you to authenticate using your SSH key.

   If you decide to use public key authentication for the 'dev' user, you should disable password login for this specific user to enhance security. To do this, follow these steps:
   
   1. Create a new SSH configuration file with higher priority:
      ```
      sudo nano /etc/ssh/sshd_config.d/10-disable-passwords.conf
      ```

   2. Add the following content to disable password authentication for all users except root:
      ```
      PasswordAuthentication no

      Match User root
            PasswordAuthentication yes
      ```

   3. Save the file and exit the editor.

   4. Restart the SSH service to apply the changes:
      ```
      sudo systemctl restart ssh
      ```

   Why: Disabling password authentication for the 'dev' user ensures that only users with the correct SSH key can log in, significantly reducing the risk of brute-force attacks. Creating a separate configuration file with a higher priority ensures that your custom settings are applied without modifying the main SSH configuration file. This approach is cleaner and makes it easier to manage SSH settings. 

2. Create an app-specific user: *(Choose name as your app)*
   ```
   sudo adduser app-user --disabled-password
   ```
   Why: Using an app-specific user further isolates the application, minimizing potential damage if the account is compromised.

   Also add the user to www-data group.

   ```
   sudo usermod -aG www-data app-user
   ```

3. Switch to the dev user:
   ```
   su - dev
   ```

**From this point on, perform all operations as the 'dev' user for app specific tasks unless specified otherwise.**

## 1. Server Setup

1. Update and upgrade your Ubuntu server:
   ```
   sudo apt update && sudo apt upgrade
   ```
   Why: This ensures your system has the latest security patches and software versions.

2. Install required packages:
   ```
   sudo apt install php-fpm php-mysql php-mbstring php-xml php-curl php-cli php-bcmath php-zip php-gd php-intl -y
   ```
   > Above command also includes installing PHP, if your app requires different php version you have to adjust it.

   Why: These packages are essential for running a Laravel application.

3. Install and configure MySQL:
   ```
   sudo apt install mysql-server -y
   sudo mysql_secure_installation
   ```
   This will prompt you some questions answer it according to your needs.

   Why: This installs MySQL and runs a security script to remove insecure defaults.

4. Create a database and user for your Laravel application:
   ```
   sudo mysql
   ```
   Then in the MySQL prompt:
   ```sql
   CREATE DATABASE laravel_db;
   CREATE USER 'laravel_user'@'localhost' IDENTIFIED BY 'your_strong_password_here';
   GRANT ALL PRIVILEGES ON laravel_db.* TO 'laravel_user'@'localhost';
   FLUSH PRIVILEGES;
   EXIT;
   ```
   Why: This sets up a dedicated database and user for your Laravel application, enhancing security and organization.

4. Install Nginx
    ```
    sudo apt install nginx -y
    ```

5. Installing composer (Instructions copied from [Download Composer](https://getcomposer.org/download/))
    1. Download composer setup
        ```
        php -r "copy('https://getcomposer.org/installer', 'composer-setup.php');"
        ```
    2. Verify the installer SHA-384
        ```
        php -r "if (hash_file('sha384', 'composer-setup.php') === 'dac665fdc30fdd8ec78b38b9800061b4150413ff2e3b6f88543c636f7cd84f6db9189d43a81e5503cda447da73c7e5b6') { echo 'Installer verified'; } else { echo 'Installer corrupt'; unlink('composer-setup.php'); } echo PHP_EOL;"
        ```
    3. Install composer
        ```
        sudo php composer-setup.php --filename=composer --install-dir=/usr/local/bin
        ```



## 2. Deploy Laravel Application

1. Set up the directory structure for Envoy:
   ```
   git clone https://github.com/laravel/laravel.git
   
   sudo mkdir -p /var/www/your-app
   sudo mkdir -p /var/www/your-app/releases
   cp -Ra laravel/storage/ /var/www/your-app/storage

   rm -rf laravel
   ```
   Why: This creates a directory structure suitable for zero-downtime deployments with Laravel Envoy.
   
   *It is a good structure for the app even if you are not using envoy.*

2. Set proper permissions:
   ```
   sudo chown -R app-user:www-data /var/www/your-app
   sudo chmod -R 755 /var/www/your-app
   sudo chmod -R 775 /var/www/your-app/storage
   ```
    755 permissions for a file or directory mean that the owner has full read, write, and execute permissions, while other users and groups can only read and execute the file or directory.

   Why: This ensures the app-specific user and web server have the necessary permissions.

3. Clone your Laravel application into a new release directory:
   ```
   sudo su - dev git clone git@github.com:username-here/repo-name-here.git /var/www/your-app/releases/$(date +%Y_%m_%d_%H)
   ```
   Why: This clones your application into a timestamped directory, allowing for easy rollbacks if needed.

4. Create a symlink to the latest release:
   ```
   sudo -u dev ln -fns /var/www/your-app/releases/$(ls -t /var/www/your-app/releases | head -1) /var/www/your-app/current
   ```
   Why: This creates a symlink named 'current' pointing to the latest release, which Nginx will serve.



5. Install dependencies:
   ```
   cd /var/www/your-app/current
   sudo -u app-user composer install --no-dev
   sudo -u app-user chmod -R 775 /var/www/your-app/current/bootstrap/cache
   ```
   Why: This installs the required PHP packages for your application, excluding development dependencies.

6. Set up environment file:
   ```
   sudo -u app-user cp .env.example .env
   sudo -u app-user php artisan key:generate
   ```
   Why: This creates your .env file and generates a unique application key for encrypting data.

7. Configure your `.env` file:
   ```
   sudo -u app-user nano .env
   ```
   Update the following lines:
   ```
   APP_ENV=production
   APP_DEBUG=false
   APP_URL=http://your-domain.com

   DB_CONNECTION=mysql
   DB_HOST=127.0.0.1
   DB_PORT=3306
   DB_DATABASE=laravel_db
   DB_USERNAME=laravel_user
   DB_PASSWORD=your_strong_password

   QUEUE_CONNECTION=database
   ```
   Why: This configures your application with the correct environment settings and database credentials.

8. Run migrations:
   ```
   sudo -u app-user php artisan migrate
   ```
   Why: This sets up your database schema.

9. Optimize Laravel:
   ```
   sudo -u app-user php artisan config:cache
   sudo -u app-user php artisan route:cache
   sudo -u app-user php artisan view:cache
   ```
   Why: These commands cache configuration, routes, and views, improving application performance.

## 3. Setup Laravel Queue with Systemd (Multiple Workers)

1. Create a new systemd service file:
   ```
   sudo nano /etc/systemd/system/laravel-worker@.service
   ```

2. Add the following content:
   ```
   [Unit]
   Description=Laravel Queue Worker %I
   After=network.target

   [Service]
   User=app-user
   Group=www-data
   Restart=always
   ExecStart=/usr/bin/php /var/www/your-app/current/artisan queue:work --queue=default --sleep=3 --max-time=3600
   
   [Install]
   WantedBy=multi-user.target
   ```
   Why: This creates a template service that can be used to start multiple queue workers. The `%I` in the description will be replaced with the instance name.

3. Start and enable three worker services:
    ```
    sudo systemctl enable laravel-worker@{1..3}
    sudo systemctl start laravel-worker@{1..3}
    ```
    Why: This starts and enables three separate queue worker processes, allowing for parallel processing of queue jobs.

4. Check the status of the workers:
   ```
   sudo systemctl status laravel-worker@{1..3}
   ```
   Why: This allows you to verify that all three workers are running correctly.

For more information on Laravel queues and workers, refer to the [official Laravel documentation on queues](https://laravel.com/docs/queues).

For more details on using systemd, you can check the [systemd documentation](https://www.freedesktop.org/software/systemd/man/systemd.service.html).

## 4. Setup Laravel Scheduler (Cron Jobs)

1. Open the crontab for the app-user user:
   ```
   sudo crontab -u app-user -e
   ```

2. Add the following line to run the Laravel scheduler every minute:
   ```
   * * * * * php /var/www/your-app/artisan schedule:run >> /dev/null 2>&1
   ```
   Why: This allows Laravel to run scheduled tasks, which is necessary for many Laravel applications.


# Configure Nginx
   1. Create a new site configuration file:
   ```
   sudo nano /etc/nginx/sites-available/your-app
   ```
   2. Add the following configuration (adjust as needed, copied from laravel docs):
   ```
   server {
        listen 80;
        listen [::]:80;
        server_name example.com;
        root /srv/example.com/public;

        add_header X-Frame-Options "SAMEORIGIN";
        add_header X-Content-Type-Options "nosniff";

        index index.php;

        charset utf-8;

        location / {
            try_files $uri $uri/ /index.php?$query_string;
        }

        location = /favicon.ico { access_log off; log_not_found off; }
        location = /robots.txt  { access_log off; log_not_found off; }

        error_page 404 /index.php;

        location ~ \.php$ {
            fastcgi_pass unix:/var/run/php/php8.2-fpm.sock;
            fastcgi_param SCRIPT_FILENAME $realpath_root$fastcgi_script_name;
            include fastcgi_params;
            fastcgi_hide_header X-Powered-By;
        }

        location ~ /\.(?!well-known).* {
            deny all;
        }
    } 
   ```

Above code also specifies php socket `unix:/var/run/php/php8.2-fpm.sock` adjust it according to your php version. `php -v`

Why: This configuration tells Nginx how to handle requests for your Laravel application, including routing PHP requests to PHP-FPM.

3. Enable the site:
    ```
    sudo ln -s /etc/nginx/sites-available/your-app /etc/nginx/sites-enabled/
    ```
    Why: This creates a symlink in the 'sites-enabled' directory, activating the configuration.

4. Test and reload Nginx:
    ```
    sudo nginx -t
    sudo systemctl reload nginx
    ```
    Why: This checks for configuration errors and applies the new settings.


## Additional Steps

### Finding Your Server's Public IPv4 Address

To find your server's public IPv4 address, you can use one of the following methods:

1. From within the server (SSH into your server first):
   ```
   curl -4 icanhazip.com
   ```
   or
   ```
   dig +short myip.opendns.com @resolver1.opendns.com
   ```

2. From your VPS provider's control panel:
   Most VPS providers display the server's IP address in their control panel or dashboard.

Why: You need the server's public IP address to configure DNS settings and to connect to your server.

### Pointing a Subdomain to Your Server

To point a subdomain to your server:

1. Log in to your domain registrar's website or DNS management interface.

2. Create a new A record:
   - Type: A
   - Name: Your subdomain (e.g., 'app' for app.yourdomain.com)
   - Value: Your server's public IPv4 address
   - TTL: Can be set to automatic or 3600 seconds (1 hour)

3. Save the changes and wait for DNS propagation (can take up to 48 hours, but often much quicker).

Why: This tells the internet to direct traffic for your subdomain to your server's IP address.

To verify DNS propagation:
```
dig +short your-subdomain.your-domain.com
```
This should return your server's IP address once propagation is complete.

### Updating Nginx Configuration

After setting up your subdomain, update your Nginx configuration to recognize it:

1. Edit your Nginx configuration file:
   ```
   sudo nano /etc/nginx/sites-available/your-app
   ```

2. Update the `server_name` directive:
   ```
   server_name your-subdomain.your-domain.com;
   ```

3. Test and reload Nginx:
   ```
   sudo nginx -t
   sudo systemctl reload nginx
   ```

Why: This ensures Nginx correctly handles requests to your subdomain.

## Install SSL Certificate Using Certbot

After setting up your domain and Nginx configuration, it's crucial to secure your site with HTTPS. We'll use Certbot to obtain and install a free SSL certificate from Let's Encrypt.

1. Install Certbot and its Nginx plugin:
   ```
   sudo apt update
   sudo apt install certbot python3-certbot-nginx -y
   ```
   Why: Certbot automates the process of obtaining and installing SSL certificates.

2. Obtain and install the certificate:
   ```
   sudo certbot --nginx -d your-subdomain.your-domain.com
   ```
   Replace `your-subdomain.your-domain.com` with your actual domain.

   Why: This command tells Certbot to use the Nginx plugin, obtain an SSL certificate, and automatically configure Nginx to use it.

3. Follow the prompts:
   - Enter your email address for important notifications.
   - Agree to the terms of service.
   - Choose whether to redirect HTTP traffic to HTTPS (recommended for most cases).

4. Verify the Nginx configuration:
   ```
   sudo nginx -t
   sudo systemctl reload nginx
   ```
   Why: This ensures that the new SSL configuration is correct and applies the changes.

5. Test automatic renewal:
   ```
   sudo certbot renew --dry-run
   ```
   Why: Certbot sets up a renewal timer, but this test ensures it's working correctly.

After successful installation, your Nginx configuration will be updated to use the new SSL certificate, and your site will be accessible via HTTPS.

### Update Laravel Environment

Now that you have HTTPS set up, update your Laravel `.env` file to use HTTPS:

```
APP_URL=https://your-subdomain.your-domain.com
```

Also, if you're using any Laravel features that generate URLs (like queue workers or scheduled tasks), you may need to set the following:

```
ASSET_URL=https://your-subdomain.your-domain.com
```

Why: This ensures that Laravel generates HTTPS URLs when needed.

## Additional Resources

- [Laravel Deployment Best Practices](https://laravel.com/docs/deployment)
- [Nginx Documentation](https://nginx.org/en/docs/)
- [Ubuntu Server Guide](https://ubuntu.com/server/docs)
- [PHP-FPM Configuration](https://www.php.net/manual/en/install.fpm.configuration.php)
- [MySQL Documentation](https://dev.mysql.com/doc/)
- [Laravel Envoy Documentation](https://laravel.com/docs/envoy)
- [DigitalOcean's DNS documentation](https://docs.digitalocean.com/products/networking/dns/) (useful even if you're not using DigitalOcean)
- [Nginx Server Names documentation](https://nginx.org/en/docs/http/server_names.html)
- [Certbot Documentation](https://certbot.eff.org/docs/)
- [Let's Encrypt Documentation](https://letsencrypt.org/docs/)
- [SSL Labs Server Test](https://www.ssllabs.com/ssltest/) - Use this to check your SSL configuration

Remember that Let's Encrypt certificates are valid for 90 days, but Certbot sets up an automatic renewal process. You can manually renew the certificate at any time using:

```
sudo certbot renew
```

Ensure that your firewall allows HTTPS traffic (usually port 443) if you have one configured.
