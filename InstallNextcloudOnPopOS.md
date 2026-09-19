# Install Nextcloud on Pop!_OS

Pop!_OS is primarily a desktop operating system. It can host a small Nextcloud instance, but Ubuntu Server or Debian is usually simpler for a dedicated server.

1. Install a currently supported Pop!_OS release and apply all updates.
2. In **Settings > Power**, disable automatic suspend while plugged in.
3. Give the computer a stable hostname in **Settings > About**.
4. Follow the shared [Nextcloud installation procedure](InstallNextcloud.md).

The shared procedure uses unversioned `php-*` package names so APT selects the PHP version supplied by the operating system. Before installing Nextcloud, compare `php --version` with the [current Nextcloud requirements](https://docs.nextcloud.com/server/stable/admin_manual/installation/system_requirements.html). Avoid an unofficial PHP PPA unless the supplied PHP is unsupported and you are prepared to maintain that repository.
