### Install Nextcloud on Ubuntu 24.04

Install Ubuntu 24.04:

Download the Ubuntu iso from here: https://releases.ubuntu.com/24.04/ubuntu-24.04-live-server-amd64.iso

Write it to USB thumb drive with something like Balena Etcher

Boot to it

Go through the installer like usual

Reboot into the new OS

Install required packages:

```
sudo apt update && sudo apt install -y needrestart needrestart-session mariadb-server php php-apcu php-bcmath php-cli php-common php-curl php-gd php-gmp php-imagick php-intl php-mbstring php-mysql php-zip php-xml php-imap vim git build-essential redis-server php-dev ffmpeg ssh fish apt-transport-https && sudo apt upgrade -y && sudo apt autoremove -y && sudo apt autoclean -y && sudo systemctl enable --now ssh && chsh -s /usr/bin/fish && fish 
```

configure fish:
```
fish_config theme choose "Bay Cruise" && fish_config theme save && fish_config prompt choose terlar && fish_config prompt save
```

```
sudo -i
```

```
chsh -s /usr/bin/fish && fish
```

configure fish:

```
fish_config theme choose "Bay Cruise" && fish_config theme save && fish_config prompt choose terlar && fish_config prompt save
```

exit

Create alias to use occ command:

```
echo 'function occ
sudo -u www-data php occ $argv
end' | tee -a ~/.config/fish/config.fish
```

```
source ~/.config/fish/config.fish
```

enable fix for occ commands:

```
php --ini
sudo vi /etc/php/8.3/cli/php.ini
```

add this to the bottom:

```
apc.enable_cli=1
```

verify the changes:

```
php -r 'echo ini_get("apc.enable_cli");'
```

```
sudo reboot
```

Setup database server:

```
sudo mysql_secure_installation
```

Follow the prompts to set up some very basic security defaults for the database server

```
sudo mariadb -u root -p
```

```
CREATE DATABASE IF NOT EXISTS nextcloud CHARACTER SET utf8mb4 COLLATE utf8mb4_general_ci;
SHOW DATABASES;
GRANT ALL PRIVILEGES ON nextcloud.* TO 'nextcloud'@'localhost' IDENTIFIED BY 'SOMECOOLPASSWORD';
FLUSH PRIVILEGES;
quit;
```

Configure PHP:

```
sudo sed -i 's/memory_limit = 128/memory_limit = 512M/g; s/upload_max_filesize = 2M/upload_max_filesize = 900M/g; s/max_execution_time = 30/max_execution_time = 360/g; s/post_max_size = 8M/post_max_size = 900M/g; s/date.timezone =/date.timezone = America\/Chicago/g; s/opcache.enable=1/opcache.enable=1/g; s/opcache.enable_cli=0/opcache.enable_cli=1/g; s/opcache.interned_strings_buffer=8/opcache.interned_strings_buffer=20/g; s/opcache.max_accelerated_files=10000/opcache.max_accelerated_files=20000/g; s/opcache.memory_consumption=128/opcache.memory_consumption=128/g; s/opcache.save_comments=1/opcache.save_comments=1/g; s/opcache.revalidate_freq=2/opcache.revalidate_freq=60/g; s/opcache.validate_timestamps=1/opcache.validate_timestamps=1/g' /etc/php/8.3/apache2/php.ini
```

```
sudo sed -i 's/memory_limit = -1/memory_limit = 512M/g; s/upload_max_filesize = 2M/upload_max_filesize = 900M/g; s/max_execution_time = 30/max_execution_time = 360/g; s/post_max_size = 8M/post_max_size = 900M/g; s/;date.timezone =/date.timezone = America\/Chicago/g; s/;opcache.enable=1/opcache.enable=1/g; s/;opcache.enable_cli=0/opcache.enable_cli=1/g; s/;opcache.interned_strings_buffer=8/opcache.interned_strings_buffer=20/g; s/;opcache.max_accelerated_files=10000/opcache.max_accelerated_files=20000/g; s/;opcache.memory_consumption=128/opcache.memory_consumption=128/g; s/;opcache.save_comments=1/opcache.save_comments=1/g; s/;opcache.revalidate_freq=2/opcache.revalidate_freq=60/g; s/;opcache.validate_timestamps=1/opcache.validate_timestamps=1/g' /etc/php/8.3/cli/php.ini
```

Install PHP mods:

```
sudo phpenmod apcu bcmath gmp imagick intl imap redis
sudo a2enmod rewrite headers env dir mime ssl
sudo systemctl restart apache2
```

Setup Redis Server:

```
sudo sed -i 's/# supervised auto/supervised systemd/' /etc/redis/redis.conf
sudo systemctl restart redis
```

Download and setup Nextcloud:

```
wget https://download.nextcloud.com/server/releases/latest.zipunzip latest.zip && sudo chown -R www-data:www-data nextcloud && sudo mv nextcloud /var/www && sudo a2dissite 000-default.conf && sudo systemctl reload apache2
```

create automatic setup file:

```
echo '<?php
$AUTOCONFIG = array(
"dbtype"        => "mysql",
"dbname"        => "nextcloud",
"dbuser"        => "nextcloud",
"dbpass"        => "SOMECOOLPASSWORD",
"dbhost"        => "localhost",
"adminlogin"    => "NAME",
"adminpass"     => "SOMECOOLPASSWORD",
"directory"     => "/var/www/nextcloud/data",
);' | sudo tee /var/www/nextcloud/config/autoconfig.php
```

Add server name to apache config to fix errors:

```
sudo vi /etc/apache2/apache2.conf
```

add to end:

```
ServerName nextcloud
```

Add apache virtual host for Nextcloud:

```
echo '<VirtualHost *:80>
DocumentRoot "/var/www/nextcloud/"
ServerName nextcloud

<Directory "/var/www/nextcloud/">     Require all granted     AllowOverride All     Options FollowSymLinks MultiViews </Directory>  <IfModule mod_dav.c>   Dav off </IfModule>  ErrorLog ${APACHE_LOG_DIR}/error.log CustomLog ${APACHE_LOG_DIR}/access.log combined

</VirtualHost>' | sudo tee /etc/apache2/sites-available/nextcloud.conf > /dev/null
```

Install the nextcloud site:

```
sudo a2ensite nextcloud.conf
sudo systemctl reload apache2
sudo apache2ctl configtest
```

Configure Nextcloud:

Browse to the Nextcloud server in your browser and update the configuration to match the database info you’ve used earlier.
Confirm installation of recommended apps.

additional apps to install within Nextcloud:
Client Push
[Draw.io](http://Draw.io)
EPUB Viewer
Memories
News
Preview Generator
Recognize

Enable memory caching:

```
sudo vim /var/www/nextcloud/config/config.php
```

Add the following:

```
'memcache.local' => '\OC\Memcache\APCu',
'memcache.distributed' => '\OC\Memcache\Redis',
'memcache.locking' => '\OC\Memcache\Redis',
'redis' =>
array (
'host' => 'localhost',
'port' => 6379,
'timeout' => 1.5,
'read_timeout' => 1.5,
'dbindex' => 0,
),
```

Correct permissions of config.php:

```
sudo chmod 660 /var/www/nextcloud/config/config.php
```

```
sudo systemctl reload apache2
```

Do some occ stuff:

```
cd /var/www/nextcloud
occ config:system:set htaccess.RewriteBase --value="/"
occ maintenance:update:htaccess
occ config:system:set default_phone_region --value=US
occ config:system:set maintenance_window_start --type=integer --value=1
occ config:app:set --value=yes serverinfo phpinfo
occ config:system:set default_locale --value=en_US
occ db:add-missing-primary-keys
occ db:add-missing-columns
occ db:add-missing-indices
occ maintenance:repair
occ setupchecks
```

get things ready for Memories & Recognize:

```
sudo ls -l /dev/dri/
sudo usermod -aG render www-data
sudo chmod 666 /dev/dri/renderD128
occ memories:places-setup
occ recognize:download-models
```

Setup crontab:

```
sudo crontab -u www-data -e
```

```
*/5  *  *  *  * php -f /var/www/nextcloud/cron.php
```

```
sudo systemctl reload apache2
```

finish setting up apps within Nextcloud website

#### Setup SSL:

Install tailscale:

```
curl -fsSL https://tailscale.com/install.sh | sh
```

When using SSL, take special note of the ServerName. You should specify one in the server configuration, as well as in the CommonName field of the certificate. If you want your Nextcloud to be reachable via the internet, then set both of these to the domain you want to reach your Nextcloud server.

Create Tailscale certificate:

```
sudo tailscale cert machine NAME.TAILSCALENETWORK.ts.net
```

Install FPM to handle https:

```
sudo apachectl stop
sudo apt install php8.3-fpm
sudo a2enmod proxy_fcgi setenvif
sudo a2enconf php8.3-fpm
sudo a2dismod php8.3
sudo a2dismod mpm_prefork
sudo a2enmod mpm_event
sudo a2enmod http2
sudo sed -i 's/;clear_env = no/clear_env = no/g' /etc/php/8.3/fpm/pool.d/www.conf
sudo apachectl start
```

Modify nextcloud.conf to enable https:

```
sudo vi /etc/apache2/sites-available/nextcloud.conf
```

change the <VirtualHost :80> section to:

```
ServerName NAME.TAILSCALENETWORK.ts.net
Redirect permanent / https://NAME.TAILSCALENETWORK.ts.net
RewriteCond %{REQUEST_URI} !^/.well-known/acme-challenge/
RewriteRule ^(.)$ https://%{HTTP_HOST}$1 [R=301,L]
```

add this below the *:80 VirtualHost:

```
<VirtualHost *:443>
DocumentRoot "/var/www/nextcloud/"
ServerName [NAME.TAILSCALENETWORK.ts.net](http://NAME.TAILSCALENETWORK.ts.net)

SSLEngine on SSLCertificateFile /var/lib/tailscale/certs/NAME.TAILSCALENETWORK.ts.net.crt SSLCertificateKeyFile /var/lib/tailscale/certs/NAME.TAILSCALENETWORK.ts.net.key  # enable HTTP/2, if available Protocols h2 http/1.1  # HTTP Strict Transport Security (mod_headers is required) (63072000 seconds) Header always set Strict-Transport-Security "max-age=63072000; includeSubDomains"  <Directory "/var/www/nextcloud/">     Options MultiViews FollowSymlinks     AllowOverride All     Order allow,deny     Allow from all  <IfModule mod_dav.c>   Dav off </IfModule>

</Directory>

TransferLog /var/log/apache2/nextcloud_access.log ErrorLog /var/log/apache2/nextcloud_error.log

</VirtualHost>

SSLProtocol all -SSLv3 -TLSv1 -TLSv1.1
SSLCipherSuite          ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256:ECDHE-ECDSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-GCM-SHA384:ECDHE-ECDSA-CHACHA20-POLY1305:ECDHE-RSA-CHACHA20-POLY1305:DHE-RSA-AES128-GCM-SHA256:DHE-RSA-AES256-GCM-SHA384:DHE-RSA-CHACHA20-POLY1305
SSLHonorCipherOrder     off
SSLSessionTickets       off

SSLUseStapling On
SSLStaplingCache "shmcb:logs/ssl_stapling(32768)"
```

Change apache config to reflect tailscale network:

```
sudo vi /etc/apache2/apache2.conf
```

add to end:

```
ServerName NAME.TAILSCALENETWORK.ts.net
```

Enable https in nextcloud:

```
occ config:system:set overwriteprotocol --value=https
```

Make recommended changes to FPM's php.ini:

```
sudo sed -i 's/memory_limit = 128M/memory_limit = 512M/g; s/upload_max_filesize = 2M/upload_max_filesize = 900M/g; s/max_execution_time = 30/max_execution_time = 360/g; s/post_max_size = 8M/post_max_size = 900M/g; s/;date.timezone =/date.timezone = America/Chicago/g; s/;opcache.enable=1/opcache.enable=1/g; s/;opcache.enable_cli=0/opcache.enable_cli=1/g; s/;opcache.interned_strings_buffer=8/opcache.interned_strings_buffer=20/g; s/;opcache.max_accelerated_files=10000/opcache.max_accelerated_files=20000/g; s/;opcache.memory_consumption=128/opcache.memory_consumption=128/g; s/;opcache.save_comments=1/opcache.save_comments=1/g; s/;opcache.revalidate_freq=2/opcache.revalidate_freq=60/g; s/;opcache.validate_timestamps=1/opcache.validate_timestamps=1/g' /etc/php/8.3/fpm/php.ini
```

```
sudo apache2ctl configtest
sudo service apache2 restart
```

Perform necessary occ commands for https changes:

```
occ config:system:set trusted_domains 1 --value=NAME.TAILSCALENETWORK.ts.net
occ config:system:set overwrite.cli.url --value="https://localhost"
occ maintenance:repair
occ setupchecks
```

some check in case of errors:

```
sudo -u www-data php8.3 -i | grep memory_limit
sudo -u www-data php8.3 -i | grep upload_max_filesize
sudo systemctl reload php8.3-fpm.service
sudo systemctl reload apache2
```

if the php files look ok and restarting apache2 doesn't clear errors, then reboot.

if Collabora messes up after enabling https, choose own server and save.

setup Client push:

```
occ notify_push:setup
```

```
sudo reboot
```
