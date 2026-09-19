# Install Nextcloud on Debian 13

Debian 13 (Trixie) is the current stable Debian release and is supported by the current Nextcloud release. Its default PHP 8.4 packages are supported.

1. Download the current Debian 13 `amd64` netinst image from the [official download page](https://www.debian.org/download).
2. Verify its checksum, write it to a USB drive, and install Debian.
3. Select **SSH server** and **standard system utilities**. A desktop environment is optional.
4. If your normal account lacks `sudo` access, log in as `root` once and run:

   ```console
   apt update
   apt install sudo
   usermod -aG sudo USERNAME
   ```

5. Log out and back in so the group change takes effect.
6. Follow the shared [Nextcloud installation procedure](InstallNextcloud.md).

The download link points to Debian's current-release page rather than a point-release ISO that quickly becomes outdated.
