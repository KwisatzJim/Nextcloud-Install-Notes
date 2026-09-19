# Install Nextcloud on Ubuntu 24.04 LTS

Ubuntu 24.04 LTS is supported by the current Nextcloud release and supplies supported PHP 8.3 packages.

1. Download the current Ubuntu 24.04 LTS server image from the [official releases page](https://releases.ubuntu.com/24.04/).
2. Verify its checksum, write it to a USB drive, and install Ubuntu Server.
3. Select the OpenSSH server option if you want to administer the computer remotely.
4. After the first login, follow the shared [Nextcloud installation procedure](InstallNextcloud.md).

Do not add a third-party PHP repository for this procedure. Ubuntu 24.04's PHP packages meet Nextcloud 34's requirements.
