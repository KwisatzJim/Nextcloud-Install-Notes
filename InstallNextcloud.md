# Manual Nextcloud installation

This procedure installs the current production release of Nextcloud with Apache, MariaDB, PHP, APCu, and Redis. It applies to the supported Debian-family systems linked from the [README](README.md).

Replace placeholders such as `PHP_VERSION`, `NEXTCLOUD_DB_PASSWORD`, `SERVER_IP_ADDRESS`, and `HOSTNAME.TAILNET.ts.net` with real values. Run commands as your normal user unless they begin with `sudo`.

## 1. Update the operating system

```console
sudo apt update
sudo apt full-upgrade -y
sudo apt autoremove -y
sudo reboot
```

Reconnect after the reboot.

## 2. Install packages

```console
sudo apt install -y apache2 mariadb-server redis-server cron unzip wget curl \
  ca-certificates imagemagick ffmpeg libapache2-mod-php php php-cli php-apcu \
  php-bcmath php-bz2 php-curl php-exif php-gd php-gmp php-imagick php-intl \
  php-mbstring php-mysql php-redis php-xml php-zip
```

The old `php-json`, `php-date`, `php-iconv`, `php-fileinfo`, and `php-posix` package names are absent because those features are built into current PHP packages. Memcached is omitted because this procedure uses Redis.

```console
sudo systemctl enable --now apache2 mariadb redis-server cron
sudo a2enmod rewrite headers env dir mime ssl
sudo systemctl restart apache2
php --version
mariadb --version
redis-server --version
```

Compare these versions with the [current Nextcloud system requirements](https://docs.nextcloud.com/server/stable/admin_manual/installation/system_requirements.html).

## 3. Configure PHP

Find the active PHP version and configuration directory:

```console
php --ini
```

Replace `PHP_VERSION` below with the directory shown by that command, such as `8.3` or `8.4`:

```console
sudoedit /etc/php/PHP_VERSION/mods-available/nextcloud.ini
```

Add:

```ini
memory_limit = 512M
upload_max_filesize = 900M
post_max_size = 900M
max_execution_time = 360
date.timezone = America/Chicago
apc.enable_cli = 1
opcache.interned_strings_buffer = 16
```

Change the time zone if necessary, then enable the settings:

```console
sudo phpenmod nextcloud
sudo systemctl restart apache2
php -r 'echo "APCu for CLI: ", ini_get("apc.enable_cli"), PHP_EOL;'
```

The last command should print `APCu for CLI: 1`.

## 4. Create the database

```console
sudo mariadb-secure-installation
sudo mariadb
```

At the `MariaDB>` prompt, replace `NEXTCLOUD_DB_PASSWORD` with a long, unique password, then run:

```sql
CREATE DATABASE nextcloud CHARACTER SET utf8mb4 COLLATE utf8mb4_general_ci;
CREATE USER 'nextcloud'@'localhost' IDENTIFIED BY 'NEXTCLOUD_DB_PASSWORD';
GRANT ALL PRIVILEGES ON nextcloud.* TO 'nextcloud'@'localhost';
FLUSH PRIVILEGES;
EXIT;
```

Do not reuse the Nextcloud administrator password as the database password.

## 5. Download and verify Nextcloud

```console
cd /tmp
wget https://download.nextcloud.com/server/releases/latest.tar.bz2
wget https://download.nextcloud.com/server/releases/latest.tar.bz2.sha256
sha256sum --check latest.tar.bz2.sha256
```

Continue only if the checksum reports `OK`:

```console
tar -xjf latest.tar.bz2
sudo mv nextcloud /var/www/nextcloud
sudo chown -R www-data:www-data /var/www/nextcloud
```

If `/var/www/nextcloud` already exists, stop. This fresh-installation procedure must not overwrite an existing installation.

## 6. Configure Apache

```console
sudoedit /etc/apache2/sites-available/nextcloud.conf
```

Add:

```apache
<VirtualHost *:80>
    ServerName nextcloud
    DocumentRoot /var/www/nextcloud

    <Directory /var/www/nextcloud>
        Require all granted
        AllowOverride All
        Options FollowSymLinks MultiViews

        <IfModule mod_dav.c>
            Dav off
        </IfModule>
    </Directory>

    ErrorLog ${APACHE_LOG_DIR}/nextcloud_error.log
    CustomLog ${APACHE_LOG_DIR}/nextcloud_access.log combined
</VirtualHost>
```

```console
sudo a2ensite nextcloud.conf
sudo a2dissite 000-default.conf
sudo apache2ctl configtest
sudo systemctl reload apache2
```

Do not continue unless the configuration test reports `Syntax OK`.

## 7. Complete setup in a browser

Open `http://SERVER_IP_ADDRESS/` from another computer on the same network. Enter:

- A new administrator name and strong password.
- `/var/www/nextcloud/data` as the data folder, or a dedicated path owned by `www-data`.
- `nextcloud` as both the database user and database name.
- The database password created above.
- `localhost` as the database host.

Finish the installation and install the recommended apps if desired.

## 8. Configure Redis and APCu

Nextcloud recommends a Unix socket when Redis is on the same computer:

```console
sudo usermod -aG redis www-data
sudoedit /etc/redis/redis.conf
```

Set or uncomment these lines in `/etc/redis/redis.conf`:

```ini
unixsocket /run/redis/redis-server.sock
unixsocketperm 770
```

```console
sudo systemctl restart redis-server apache2
sudo -u www-data php /var/www/nextcloud/occ config:system:set memcache.local --value='\OC\Memcache\APCu'
sudo -u www-data php /var/www/nextcloud/occ config:system:set memcache.distributed --value='\OC\Memcache\Redis'
sudo -u www-data php /var/www/nextcloud/occ config:system:set memcache.locking --value='\OC\Memcache\Redis'
sudo -u www-data php /var/www/nextcloud/occ config:system:set redis host --value='/run/redis/redis-server.sock'
sudo -u www-data php /var/www/nextcloud/occ config:system:set redis port --type=integer --value=0
```

## 9. Configure background jobs and defaults

```console
echo '*/5 * * * * php -f /var/www/nextcloud/cron.php' | sudo crontab -u www-data -
sudo -u www-data php /var/www/nextcloud/occ background:cron
sudo -u www-data php /var/www/nextcloud/occ config:system:set default_phone_region --value=US
sudo -u www-data php /var/www/nextcloud/occ config:system:set default_locale --value=en_US
sudo -u www-data php /var/www/nextcloud/occ config:system:set maintenance_window_start --type=integer --value=1
sudo -u www-data php /var/www/nextcloud/occ maintenance:update:htaccess
sudo -u www-data php /var/www/nextcloud/occ setupchecks
```

`maintenance_window_start` uses UTC; change `1` if 01:00-05:00 UTC is not a low-usage period. The full `occ` command works from any directory and follows the system's active PHP version.

## 10. Optional apps

Install only needed apps from Nextcloud's Apps page. Examples include Client Push, Draw.io, EPUB Viewer, Memories, News, Preview Generator, and Recognize. Confirm app compatibility before a major Nextcloud upgrade.

Memories and Recognize have separate hardware-acceleration instructions. Do not make `/dev/dri/renderD128` world-writable with `chmod 666`; follow each app's documentation and grant the minimum required group access.

## 11. Optional Tailscale HTTPS

This exposes Nextcloud only to devices allowed by the tailnet policy. First enable MagicDNS and HTTPS certificates in the Tailscale admin console. The machine's complete `*.ts.net` name will be published in public Certificate Transparency logs.

```console
curl -fsSL https://tailscale.com/install.sh | sh
sudo tailscale up
```

Replace `HOSTNAME.TAILNET.ts.net` with this machine's complete MagicDNS name:

```console
sudo install -d -m 750 -o root -g www-data /etc/apache2/tailscale-certs
sudo tailscale cert \
  --cert-file=/etc/apache2/tailscale-certs/nextcloud.crt \
  --key-file=/etc/apache2/tailscale-certs/nextcloud.key \
  HOSTNAME.TAILNET.ts.net
sudo chmod 640 /etc/apache2/tailscale-certs/nextcloud.key
sudo a2enmod http2
sudoedit /etc/apache2/sites-available/nextcloud.conf
```

Replace that file's contents with:

```apache
<VirtualHost *:80>
    ServerName HOSTNAME.TAILNET.ts.net
    Redirect permanent / https://HOSTNAME.TAILNET.ts.net/
</VirtualHost>

<VirtualHost *:443>
    ServerName HOSTNAME.TAILNET.ts.net
    DocumentRoot /var/www/nextcloud

    SSLEngine on
    SSLCertificateFile /etc/apache2/tailscale-certs/nextcloud.crt
    SSLCertificateKeyFile /etc/apache2/tailscale-certs/nextcloud.key
    Protocols h2 http/1.1
    Header always set Strict-Transport-Security "max-age=15552000"

    <Directory /var/www/nextcloud>
        Require all granted
        AllowOverride All
        Options FollowSymLinks MultiViews

        <IfModule mod_dav.c>
            Dav off
        </IfModule>
    </Directory>

    ErrorLog ${APACHE_LOG_DIR}/nextcloud_error.log
    CustomLog ${APACHE_LOG_DIR}/nextcloud_access.log combined
</VirtualHost>
```

Test Apache before reloading it, then configure Nextcloud's canonical URL:

```console
sudo apache2ctl configtest
sudo systemctl reload apache2
sudo -u www-data php /var/www/nextcloud/occ config:system:set trusted_domains 1 --value=HOSTNAME.TAILNET.ts.net
sudo -u www-data php /var/www/nextcloud/occ config:system:set overwrite.cli.url --value=https://HOSTNAME.TAILNET.ts.net
sudo -u www-data php /var/www/nextcloud/occ config:system:set overwriteprotocol --value=https
sudo -u www-data php /var/www/nextcloud/occ setupchecks
```

Tailscale certificates expire after 90 days. File-based certificates are not renewed automatically. Renew both files with the same `tailscale cert` command before expiration, then reload Apache.

## 12. Final checks

```console
sudo apache2ctl configtest
sudo systemctl --no-pager --full status apache2 mariadb redis-server cron
sudo -u www-data php /var/www/nextcloud/occ status
sudo -u www-data php /var/www/nextcloud/occ setupchecks
sudo crontab -u www-data -l
```

Review **Administration settings > Overview** in Nextcloud and resolve its warnings before relying on the server.

## References

- [Nextcloud system requirements](https://docs.nextcloud.com/server/stable/admin_manual/installation/system_requirements.html)
- [Nextcloud Linux installation](https://docs.nextcloud.com/server/stable/admin_manual/installation/source_installation.html)
- [Nextcloud PHP configuration](https://docs.nextcloud.com/server/stable/admin_manual/installation/php_configuration.html)
- [Nextcloud memory caching](https://docs.nextcloud.com/server/stable/admin_manual/configuration_server/caching_configuration.html)
- [Nextcloud background jobs](https://docs.nextcloud.com/server/stable/admin_manual/configuration_server/background_jobs_configuration.html)
- [Tailscale HTTPS certificates](https://tailscale.com/docs/how-to/set-up-https-certificates)
