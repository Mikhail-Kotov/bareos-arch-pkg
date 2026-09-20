# Quickstart
```
git clone git@github.com:Mikhail-Kotov/bareos-arch-pkg.git
```

To install using PKGBUILD:
```
cd bareos-arch-pkg/bareos
makepkg -si

cd ../bareos-webui
makepkg -si
```

Or strait away, if you don't want to compile, using pacman.
```
cd bareos-arch-pkg/bareos
pacman -U bareos-25.1.1-1-x86_64.pkg.tar.zst
cd ../bareos-webui
pacman -U bareos-webui-25.1.1-1-any.pkg.tar.zst
```

# Bareos 25.1.1 PKGBUILDs for Arch Linux / CachyOS
 
Self-contained PKGBUILDs that build [Bareos](https://www.bareos.com) 25.1.1 from the
upstream git tag on current Arch-based systems.
 
## Why this repository exists
 
The Bareos package in the AUR (25.0.1-2) no longer compiles on a current toolchain, and
it is flagged out of date. Rather than wait for it, these PKGBUILDs build the upstream
`Release/25.1.1` tag directly and carry the small fixes needed to make it compile,
install and run on Arch/CachyOS.
 
Built and tested on CachyOS with GCC 16.2.1, fmt 12.2.0, CMake 4.4, Python 3.14 and
PostgreSQL 18.
 
## Contents
 
| Directory | Package | Description |
|---|---|---|
| `bareos/` | `bareos` | One all-in-one package: Director, Storage Daemon, File Daemon, `bconsole`, tools, PostgreSQL catalog backend, Python plugins, systemd units, sysusers/tmpfiles. Built without tray monitor, WebUI and system tests. |
| `bareos-webui/` | `bareos-webui` | Classic PHP WebUI, packaged separately (the new Vue WebUI preview is not included). Ships an nginx example config. |
 
### Fixes applied in `bareos/PKGBUILD` (`prepare()` / `build()`)
 
- `distname.sh`: set `DISTVER=rolling` on Arch, otherwise CMake aborts (`list index: 1 out of range`).
- Drop the hard-coded `-Werror` so new compiler warnings do not break the build.
- fmt 12 compatibility: include `<fmt/format.h>`, and format `FMT_STRING()` arguments at runtime via `fmt::runtime()`.
- Add the missing `#include <cstring>` (libstdc++ 16).
- Enable the systemd unit files, which upstream skips for the `archlinux` platform.
- Pass the `XXX_REPLACE_WITH_*` password/hostname placeholders to CMake so `bareos-config` can fill in real values when deploying the configuration.
- `tmpfiles.d` creates `/run/bareos`, `/var/lib/bareos`, `/var/lib/bareos/storage` and `/var/log/bareos` owned by `bareos`.
## Installation
 
### 0. Requirements
 
- `base-devel`, `git`, `cmake`, `rpcsvc-proto`
- PostgreSQL installed, initialised and running
- Optional: `glusterfs` (enables the GlusterFS backend/plugin if present at build time)
### 1. Build and install Bareos
 
```
cd bareos
makepkg -si
```
 
### 2. Deploy the default configuration
 
This copies the templates from `/usr/share/bareos/config` into `/etc/bareos`, sets the
hostname, generates random passwords and fixes permissions. Existing configuration is
never overwritten.
 
```
sudo /usr/lib/bareos/scripts/bareos-config deploy_config bareos-dir
sudo /usr/lib/bareos/scripts/bareos-config deploy_config bareos-sd
sudo /usr/lib/bareos/scripts/bareos-config deploy_config bareos-fd
sudo /usr/lib/bareos/scripts/bareos-config deploy_config bconsole
```
 
### 3. Create the catalog database
 
Run the scripts as the `postgres` user. The database is created with the encoding
Bareos requires (SQL_ASCII, `C` locale); the `bareos` database role is created by the
last script.
 
```
sudo -u postgres /usr/lib/bareos/scripts/create_bareos_database
sudo -u postgres /usr/lib/bareos/scripts/make_bareos_tables
sudo -u postgres /usr/lib/bareos/scripts/grant_bareos_privileges
```
 
If a `bareos` catalog from an older Bareos version already exists, skip the above,
make a dump and upgrade the schema instead:
 
```
sudo -u postgres pg_dump bareos > bareos-catalog-backup.sql
sudo -u postgres /usr/lib/bareos/scripts/update_bareos_tables
```
 
### 4. Start the services
 
```
sudo systemctl enable --now bareos-dir bareos-sd bareos-fd
systemctl status bareos-dir bareos-sd bareos-fd
```
 
### 5. (Optional) Hosts without a local mail server
 
The default configuration sends the catalog bootstrap file and job reports through
`bsmtp`. Without an MTA this makes `BackupCatalog` end with an error. Write the
bootstrap file to disk and disable mail delivery:
 
```
sudo sed -i 's#^\(\s*\)Write Bootstrap = ".*#\1Write Bootstrap = "/var/lib/bareos/%n.bsr"#' /etc/bareos/bareos-dir.d/job/BackupCatalog.conf
sudo sed -i 's/^\(\s*mail = \)/#\1/; s/^\(\s*operator = \)/#\1/' /etc/bareos/bareos-dir.d/messages/Standard.conf /etc/bareos/bareos-dir.d/messages/Daemon.conf
sudo -u bareos bareos-dir -t
echo reload | sudo bconsole
```
 
### 6. Verify
 
```
sudo bconsole
*run job=backup-bareos-fd yes
*wait
*messages
*run job=BackupCatalog yes
*wait
*messages
```
 
Both jobs should finish with `Termination: Backup OK`. To test a restore, run `restore`,
choose option 5 (most recent backup), select the client, mark a single file, and confirm
that `Where:` is `/tmp/bareos-restores`. Then compare the restored file with the original
using `cmp`.
 
The default jobs only back up `/usr/bin` and the catalog, and write volumes to
`/var/lib/bareos/storage`. Define your own FileSet, Storage and Schedule before relying
on it.
 
## Optional: WebUI
 
### 7. Build and install the WebUI
 
```
cd bareos-webui
makepkg -si
```
 
### 8. Enable the required PHP extensions
 
```
sudo tee /etc/php/conf.d/bareos-webui.ini <<'EOF'
extension=bz2
extension=gettext
extension=gd
extension=iconv
extension=intl
EOF
sudo systemctl restart php-fpm
```
 
Check that nothing is missing (the command should print nothing):
 
```
for m in bz2 ctype curl date dom fileinfo filter gettext gd hash iconv intl json mbstring openssl pcre reflection session; do php -m | tr A-Z a-z | grep -qx "$m" || echo "missing: $m"; done
```
 
### 9. Create the WebUI console user in the Director
 
The PHP WebUI does not support TLS-PSK, so the example console disables TLS. Only use
this when the WebUI and the Director run on the same host.
 
```
PW=$(openssl rand -base64 18 | tr -d '/+=')
sudo cp /usr/share/bareos/config/bareos-dir.d/console/admin.conf.example /etc/bareos/bareos-dir.d/console/admin.conf
sudo sed -i "s/Password = \"admin\"/Password = \"$PW\"/" /etc/bareos/bareos-dir.d/console/admin.conf
sudo chown --reference=/etc/bareos/bareos-dir.d/console/bareos-mon.conf /etc/bareos/bareos-dir.d/console/admin.conf
sudo chmod --reference=/etc/bareos/bareos-dir.d/console/bareos-mon.conf /etc/bareos/bareos-dir.d/console/admin.conf
echo "$PW"
sudo -u bareos bareos-dir -t
echo reload | sudo bconsole
```
 
Save the printed password: it is the WebUI password for the user `admin`.
 
### 10. Configure nginx and php-fpm
 
```
sudo pacman -S --needed nginx
sudo mkdir -p /etc/nginx/conf.d
sudo cp /usr/share/doc/bareos-webui/nginx-bareos-webui.conf /etc/nginx/conf.d/bareos-webui.conf
```
 
Make sure the `http { }` block of `/etc/nginx/nginx.conf` contains:
 
```
include /etc/nginx/conf.d/*.conf;
```
 
Then:
 
```
sudo nginx -t
sudo systemctl enable --now php-fpm nginx
```
 
Open `http://localhost:9100` and log in as `admin`. The site is served over plain HTTP;
put TLS in front of it before exposing it beyond localhost.
 
## License
 
The PKGBUILDs are provided as-is under Apache-2.0. Bareos itself is licensed under AGPL-3.0.
