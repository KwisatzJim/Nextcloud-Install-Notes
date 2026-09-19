# Install Nextcloud on AlmaLinux 10

This procedure installs the current production release of Nextcloud on AlmaLinux 10 with Apache, PHP-FPM, MariaDB, APCu, and Valkey. AlmaLinux 10 follows Red Hat Enterprise Linux 10 closely; RHEL 10 is a recommended platform in the current Nextcloud requirements.

The procedure was reviewed on September 19, 2026. It is for a fresh installation, not an in-place upgrade. Replace placeholders such as `NEXTCLOUD_DB_PASSWORD`, `SERVER_IP_ADDRESS`, and `HOSTNAME.TAILNET.ts.net` before running commands.

## 1. Install and update AlmaLinux

Download AlmaLinux 10 from the [official download page](https://almalinux.org/get-almalinux/), verify the checksum, and install the **Minimal Install** environment. Create a normal administrator account during installation.

After the first login:

```console
sudo dnf upgrade --refresh -y
sudo reboot
```

Reconnect after the reboot and confirm the release:

```console
cat /etc/almalinux-release
```

## 2. Install the required packages

AlmaLinux 10 supplies PHP 8.3, MariaDB 10.11, and Valkey 8.0. These versions meet Nextcloud 34's requirements. Valkey is a Redis-compatible cache server and replaces the Redis server package in this procedure.

```console
sudo dnf install -y httpd mod_ssl mariadb-server valkey cronie policycoreutils-python-utils \
  tar bzip2 wget curl ca-certificates php php-cli php-fpm php-gd php-mysqlnd \
  php-curl php-mbstring php-intl php-gmp php-xml php-bcmath php-process \
  php-opcache php-pecl-zip php-pecl-apcu php-pecl-redis6
```

If DNF reports that one of the PHP packages is unavailable, stop rather than enabling an unknown third-party repository. Check the enabled repositories and package names:

```console
sudo dnf repolist
sudo dnf search php-pecl-apcu php-pecl-redis php-pecl-zip
```

Start the services and enable them at boot:

```console
sudo systemctl enable --now httpd php-fpm mariadb valkey crond
```

Confirm the installed versions:

```console
php --version
httpd -v
mariadb --version
valkey-server --version
```

## 3. Configure PHP-FPM

Create a Nextcloud-specific PHP configuration file:

```console
sudoedit /etc/php.d/99-nextcloud.ini
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

Change the time zone if necessary. Restart PHP-FPM and verify the CLI setting:

```console
sudo systemctl restart php-fpm httpd
php -r 'echo "APCu for CLI: ", ini_get("apc.enable_cli"), PHP_EOL;'
```

The last command should print `APCu for CLI: 1`.

## 4. Configure MariaDB

Run MariaDB's security helper:

```console
sudo mariadb-secure-installation
```

Create a configuration file for Nextcloud's required transaction isolation level:

```console
sudoedit /etc/my.cnf.d/nextcloud.cnf
```

Add:

```ini
[mariadb]
transaction-isolation=READ-COMMITTED
binlog_format=ROW
```

Restart MariaDB, then open its command prompt:

```console
sudo systemctl restart mariadb
sudo mariadb
```

At the `MariaDB>` prompt, replace `NEXTCLOUD_DB_PASSWORD` with a long, unique password:

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
sudo chown -R apache:apache /var/www/nextcloud
```

If `/var/www/nextcloud` already exists, stop. Do not overwrite an existing installation.

## 6. Configure SELinux

Keep SELinux in enforcing mode. Give Apache write access only to the Nextcloud locations that require it:

```console
sudo semanage fcontext -a -t httpd_sys_rw_content_t '/var/www/nextcloud/data(/.*)?'
sudo semanage fcontext -a -t httpd_sys_rw_content_t '/var/www/nextcloud/config(/.*)?'
sudo semanage fcontext -a -t httpd_sys_rw_content_t '/var/www/nextcloud/apps(/.*)?'
sudo semanage fcontext -a -t httpd_sys_rw_content_t '/var/www/nextcloud/.htaccess'
sudo semanage fcontext -a -t httpd_sys_rw_content_t '/var/www/nextcloud/.user.ini'
sudo semanage fcontext -a -t httpd_sys_rw_content_t '/var/www/nextcloud/3rdparty/aws/aws-sdk-php/src/data/logs(/.*)?'
sudo restorecon -Rv /var/www/nextcloud
```

Allow Nextcloud to reach services such as the app store and Valkey over the local network stack:

```console
sudo setsebool -P httpd_can_network_connect on
```

Do not disable SELinux to work around a permission error. Inspect denials with `sudo ausearch -m AVC -ts recent` and correct the specific label or policy instead.

## 7. Configure Apache

Create the virtual host:

```console
sudoedit /etc/httpd/conf.d/nextcloud.conf
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

    ErrorLog /var/log/httpd/nextcloud_error.log
    CustomLog /var/log/httpd/nextcloud_access.log combined
</VirtualHost>
```

Test the configuration and reload Apache:

```console
sudo apachectl configtest
sudo systemctl reload httpd
```

Do not continue unless the test reports `Syntax OK`.

## 8. Permit initial HTTP access

If `firewalld` is active, allow HTTP while performing the initial setup:

```console
sudo firewall-cmd --state
sudo firewall-cmd --permanent --add-service=http
sudo firewall-cmd --reload
```

This makes port 80 reachable through the active firewall zone. Do not expose this unencrypted setup page directly to the public internet. Use a trusted local network or Tailscale.

## 9. Complete setup in a browser

Open `http://SERVER_IP_ADDRESS/` from another computer on the trusted network. Enter:

- A new administrator name and strong password.
- `/var/www/nextcloud/data` as the data folder, or another properly labeled path owned by `apache`.
- `nextcloud` as both the database user and database name.
- The database password created above.
- `localhost` as the database host.

Finish the installation and install the recommended apps if desired.

## 10. Configure APCu and Valkey

The Valkey service listens only on the local system by default. Confirm that before continuing:

```console
sudo ss -lntp | grep 6379
```

The listening address should be `127.0.0.1:6379` or `[::1]:6379`, not every network interface.

Configure Nextcloud through `occ`:

```console
sudo -u apache php /var/www/nextcloud/occ config:system:set memcache.local --value='\OC\Memcache\APCu'
sudo -u apache php /var/www/nextcloud/occ config:system:set memcache.distributed --value='\OC\Memcache\Redis'
sudo -u apache php /var/www/nextcloud/occ config:system:set memcache.locking --value='\OC\Memcache\Redis'
sudo -u apache php /var/www/nextcloud/occ config:system:set redis host --value=127.0.0.1
sudo -u apache php /var/www/nextcloud/occ config:system:set redis port --type=integer --value=6379
```

Nextcloud's PHP cache class retains the name `Redis`; it works with the Redis-compatible Valkey server through the PHP Redis extension.

## 11. Configure background jobs and defaults

Create `/etc/cron.d/nextcloud`:

```console
sudoedit /etc/cron.d/nextcloud
```

Add exactly this line, including the `apache` user field:

```cron
*/5 * * * * apache php -f /var/www/nextcloud/cron.php
```

Apply common settings:

```console
sudo chmod 644 /etc/cron.d/nextcloud
sudo restorecon /etc/cron.d/nextcloud
sudo systemctl restart crond
sudo -u apache php /var/www/nextcloud/occ background:cron
sudo -u apache php /var/www/nextcloud/occ config:system:set default_phone_region --value=US
sudo -u apache php /var/www/nextcloud/occ config:system:set default_locale --value=en_US
sudo -u apache php /var/www/nextcloud/occ config:system:set maintenance_window_start --type=integer --value=1
sudo -u apache php /var/www/nextcloud/occ maintenance:update:htaccess
sudo -u apache php /var/www/nextcloud/occ setupchecks
```

`maintenance_window_start` uses UTC. Change `1` if 01:00-05:00 UTC is not an appropriate low-usage period.

## 12. Optional Tailscale HTTPS

Enable MagicDNS and HTTPS certificates in the Tailscale admin console first. The machine's complete `*.ts.net` hostname will appear in public Certificate Transparency logs.

Install Tailscale using its current RHEL-family repository instructions:

```console
curl -fsSL https://tailscale.com/install.sh | sh
sudo tailscale up
```

Replace `HOSTNAME.TAILNET.ts.net` with this machine's complete MagicDNS name:

```console
sudo install -d -m 750 -o root -g apache /etc/httpd/tailscale-certs
sudo tailscale cert \
  --cert-file=/etc/httpd/tailscale-certs/nextcloud.crt \
  --key-file=/etc/httpd/tailscale-certs/nextcloud.key \
  HOSTNAME.TAILNET.ts.net
sudo chown root:apache /etc/httpd/tailscale-certs/nextcloud.crt /etc/httpd/tailscale-certs/nextcloud.key
sudo chmod 640 /etc/httpd/tailscale-certs/nextcloud.key
sudo restorecon -Rv /etc/httpd/tailscale-certs
```

Label the certificate files so Apache can read them under SELinux:

```console
sudo semanage fcontext -a -t httpd_config_t '/etc/httpd/tailscale-certs(/.*)?'
sudo restorecon -Rv /etc/httpd/tailscale-certs
```

Replace `/etc/httpd/conf.d/nextcloud.conf` with:

```apache
<VirtualHost *:80>
    ServerName HOSTNAME.TAILNET.ts.net
    Redirect permanent / https://HOSTNAME.TAILNET.ts.net/
</VirtualHost>

<VirtualHost *:443>
    ServerName HOSTNAME.TAILNET.ts.net
    DocumentRoot /var/www/nextcloud

    SSLEngine on
    SSLCertificateFile /etc/httpd/tailscale-certs/nextcloud.crt
    SSLCertificateKeyFile /etc/httpd/tailscale-certs/nextcloud.key
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

    ErrorLog /var/log/httpd/nextcloud_error.log
    CustomLog /var/log/httpd/nextcloud_access.log combined
</VirtualHost>
```

Test Apache before reloading it, then configure Nextcloud's canonical URL:

```console
sudo apachectl configtest
sudo systemctl reload httpd
sudo -u apache php /var/www/nextcloud/occ config:system:set trusted_domains 1 --value=HOSTNAME.TAILNET.ts.net
sudo -u apache php /var/www/nextcloud/occ config:system:set overwrite.cli.url --value=https://HOSTNAME.TAILNET.ts.net
sudo -u apache php /var/www/nextcloud/occ config:system:set overwriteprotocol --value=https
sudo -u apache php /var/www/nextcloud/occ setupchecks
```

Tailscale certificates expire after 90 days. File-based certificates are not renewed automatically. Renew them with the same `tailscale cert` command before expiration, restore their SELinux contexts, and reload `httpd`.

Once HTTPS works, remove general HTTP access if it is no longer needed:

```console
sudo firewall-cmd --permanent --remove-service=http
sudo firewall-cmd --reload
```

Do not add the public `https` firewalld service for a Tailscale-only installation. Tailscale traffic uses the `tailscale0` interface rather than arriving as ordinary public HTTPS traffic.

## 13. Final checks

```console
getenforce
sudo apachectl configtest
sudo systemctl --no-pager --full status httpd php-fpm mariadb valkey crond
sudo -u apache php /var/www/nextcloud/occ status
sudo -u apache php /var/www/nextcloud/occ setupchecks
cat /etc/cron.d/nextcloud
```

`getenforce` should report `Enforcing`. Review **Administration settings > Overview** in Nextcloud and resolve its warnings before relying on the server.

## References

- [AlmaLinux 10 release notes](https://wiki.almalinux.org/release-notes/10.0)
- [Nextcloud system requirements](https://docs.nextcloud.com/server/stable/admin_manual/installation/system_requirements.html)
- [Nextcloud Linux installation](https://docs.nextcloud.com/server/stable/admin_manual/installation/source_installation.html)
- [Nextcloud PHP configuration](https://docs.nextcloud.com/server/stable/admin_manual/installation/php_configuration.html)
- [Nextcloud SELinux configuration](https://docs.nextcloud.com/server/stable/admin_manual/installation/selinux_configuration.html)
- [Nextcloud memory caching](https://docs.nextcloud.com/server/stable/admin_manual/configuration_server/caching_configuration.html)
- [Tailscale HTTPS certificates](https://tailscale.com/docs/how-to/set-up-https-certificates)
