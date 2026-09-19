# Nextcloud installation notes

These notes describe a manual Nextcloud installation for a small, self-hosted server using Apache, MariaDB, Redis, and optional Tailscale-only HTTPS access.

The procedures were reviewed on September 19, 2026. Nextcloud 34 is the current production release; Nextcloud 35 is still a release candidate. The download command uses Nextcloud's `latest` URL, so it follows the current production release rather than a beta or release candidate.

## Choose an operating system

- [Ubuntu 24.04 LTS](InstallNextcloudUbuntu24.04.md)
- [Debian 13](InstallNextcloudOnDebian.md)
- [Pop!_OS](InstallNextcloudOnPopOS.md)
- [AlmaLinux 10](InstallNextcloudOnAlmaLinux.md)

The Debian-family pages lead to the shared [installation procedure](InstallNextcloud.md). AlmaLinux has a separate procedure because its package names, Apache layout, PHP-FPM service, firewall, and SELinux configuration differ substantially.

Replace every placeholder before running a command. Back up the Nextcloud data directory, configuration, database, and encryption keys before upgrading or making major configuration changes. Check the [current system requirements](https://docs.nextcloud.com/server/stable/admin_manual/installation/system_requirements.html) before each new installation.
