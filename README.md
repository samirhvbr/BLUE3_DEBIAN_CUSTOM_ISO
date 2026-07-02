# Blue3 Debian ISO Custom

Project to generate a customized Debian ISO with automated installation via preseed and embedded Blue3 configuration files.

## 🔄 Before you start: `git pull`

**ALWAYS** check for remote updates before writing or changing anything in this repository:

```bash
git pull          # already pre-authorized (allow)
```

Working on top of an outdated base causes conflicts. Pull first, always. To just inspect beforehand: `git fetch && git status`.

## Goal

This directory holds the versionable material needed to rebuild the ISO:

- `script-iso.sh`: extracts the base ISO, injects the Blue3 files, adjusts the boot and generates the new ISO
- `blue3/preseed.cfg`: defines the automatic installation
- `blue3/`: files that go into the ISO under `/blue3` and are applied to the installed system in the `late_command`

Build artifacts, such as `isofiles/`, `*.iso` and `custom.log`, are kept out of versioning by the `.gitignore`.

## Automation Standard

To make the process as automatable as possible, the base ISO must sit in this directory with a fixed name:

```text
debian.iso
```

This avoids editing the script for each new image. It can be a netinst ISO, a DVD or another Debian variant, as long as it is renamed to `debian.iso` before the build.

The script also validates this ISO right at the start. If `debian.iso` does not exist, it aborts before the cleanup with a clear message in the terminal and in `custom.log`.

## The blue3_script.sh file

See the README_blue3_script.md file to understand how /root/blue3_script.sh works.

## Quick-Adjustment Variables

To avoid hunting for sensitive spots in the script, the user and group adjustments are concentrated at the start:

```bash
BUILD_USER="${BUILD_USER:-$USER}"
BUILD_GROUP="${BUILD_GROUP:-$USER}"
```

These variables control the `chown` applied to `isofiles/`, `blue3/` and the final generated ISO.

If another user is going to use the build, just adjust these variables at the start of the script or call it like this:

```bash
BUILD_USER=outro_usuario BUILD_GROUP=outro_grupo bash script-iso.sh
```

## Project Structure

```text
.
├── blue3/
│   ├── blue3_script.sh
│   ├── service.one_shot
│   ├── 10-uname
│   ├── 20-blue3
│   ├── bashrc
│   ├── blue3.png
│   ├── grub.cfg
│   ├── issue
│   ├── issue.net
│   ├── motd
│   ├── preseed.cfg
│   ├── sources.list
│   └── ssh/
|       ├── ipauth.conf
|       ├── keyregeneration.conf
|       ├── thosts.conf
|       ├── sshd_config
|       └── useprivilegeseparation.conf
├── script-iso.sh
├── debian.iso
├── README.md
└── isofiles/
```

## How the Build Works

The current flow is this:

1. `script-iso.sh` validates that `debian.iso` exists.
2. The script removes `isofiles/` and the previous output ISO for the day.
3. The base ISO `debian.iso` is extracted to `isofiles/`.
4. The contents of the `blue3/` folder are copied to `isofiles/blue3/`.
5. The script validates the existence of `isofiles/blue3/preseed.cfg`.
6. The installer's `initrd` is repackaged without the `speakup` and accessibility components.
7. The BIOS and UEFI boot menus receive the parameters for automatic installation using `preseed/file=/cdrom/blue3/preseed.cfg`.
8. The extracted ISO's `md5sum.txt` is recreated.
9. The new ISO is generated with a name in the format `blue3-debian-YYYYMMDD.iso`.

## What `script-iso.sh` assembles

### Expected Input

- Base ISO: `debian.iso`
- Customization directory: `blue3/`

### Structure Generated in the Build

- `isofiles/`: temporary tree with the extracted ISO
- `isofiles/blue3/`: copy of the local customization files
- `custom.log`: log of the build process
- `blue3-debian-YYYYMMDD.iso`: final generated ISO

### Changes Made by the Script

- Validates the existence of `debian.iso` before starting the cleanup
- Extracts the original ISO with `xorriso`
- Adjusts the owner and permissions of the files copied to the internal `blue3` folder
- Ensures the executable permission for `10-uname` and `20-blue3`
- Removes `speakup` and related files from `install.amd/initrd.gz` and `install.amd/gtk/initrd.gz`
- Adjusts the installer's boot to use automatic preseed and disable speech
- Recalculates `md5sum.txt`
- Generates the final ISO in hybrid BIOS/UEFI mode

### Injected Boot Parameters

```text
auto=true priority=critical preseed/file=/cdrom/blue3/preseed.cfg vga=788 \
debian-installer/speech=false speakup.synth=none speakup.synth=off \
debian-installer/disable-speech=true noaccessibility DEBCONF_DEBUG=5
```

## What `blue3/preseed.cfg` Configures

The `preseed.cfg` automates the installation and defines the default of the installed system.

### Installer

- Automatic installation with `debconf/priority=critical`
- `preseed/interactive=false` to reduce questions during the installation
- Speech and accessibility disabled
- Locale `pt_BR.UTF-8`
- Language `pt`
- Keyboard `us`

### Network

During the installation, automatic network configuration is blocked so as not to interfere with the process.

- Hostname: `blue3`
- Domain: `b3.local`
- IPv4: disabled
- DNS: `170.233.231.231 170.233.231.232 1.1.1.1`
- IPv6: disabled
- A default /etc/network/interfaces file is copied to make it easier to fill in later.

### Mirror and Packages

- Configured mirror: `mirror.blue3.com.br/debian`
- `apt-setup/use_mirror` is `false`
- Selected task: `standard`
- Additional packages: `sudo`, `vim`, `openssh-server`, `zstd`, `xfsprogs`, `btrfs-progs`

For truly offline use, the ideal is to use a Debian ISO that already contains the necessary packages, typically a DVD ISO renamed to `debian.iso`.

### Users

- Does not create an interactive user during the installer
- Sets the `root` user
- Sets the `samir` user as an additional sudo user
- The passwords are stored hashed inside the preseed
  - The default password can be changed by creating a new one with the command below, copying the entire result and replacing from 'password' onward
  ```text
  mkpasswd -m sha-512 "sua-senha-aqui"
  ```

Note: the file's comments indicate a default password of `blue3`. If this is kept outside a controlled environment, the ideal is to change this secret before publishing or using in production.


### Time Zone and Clock

- Timezone: `America/Sao_Paulo`
- NTP: `ntp.blue3.com.br`

### Automatic Partitioning

- Note that the purpose is a server without a graphical environment, using only the bare minimum in each partition. The remaining physical space stays free in the LVM VG and can be used later to adjust the partitions. Using /var, /log and /tmp on Btrfs allows file compression to be applied.
- Separating the /var/log directory into a dedicated partition aims to prevent excessive log-file growth from compromising the system's operation.
- Separating the /tmp directory allows security options — such as execution restrictions and other protections — to be applied directly in /etc/fstab.

- Target disk: `/dev/sda`
- Volume group: `vg0`
- Layout:
  - EFI on GPT for UEFI boot
  - `/boot` on `ext4`
  - `swap` on LVM
  - `/` on LVM `xfs`
  - `/var` on LVM `btrfs`
  - `/var/log` on LVM `btrfs`
  - `/tmp` on LVM `btrfs`

## Blue3 Files Applied in the `late_command`

At the end of the installation, the `late_command` copies files from the ISO to the installed system.

| Source on the ISO | Destination on the installed system | Purpose |
| --- | --- | --- |
| `blue3/blue3_script.sh` | `/root/blue3_script.sh` | Initial script for first-time adjustments and configuration |
| `blue3/.env` | `/root/.env` | PHPIPAM API configuration file |
| `blue3/.env.example`        | Example PHPIPAM API configuration file; configure it according to your PHPIPAM |
| `blue3/service.one_shot` | `/etc/systemd/system/blue3-firstboot.service` | Initial script forced at boot |
| `blue3/20-blue3` | `/etc/update-motd.d/20-blue3` | Dynamic Blue3 MOTD |
| `blue3/10-uname` | `/etc/update-motd.d/10-uname` | system information at login |
| `blue3/issue.net` | `/etc/issue.net` | remote banner |
| `blue3/issue` | `/etc/issue` | local banner |
| `blue3/motd` | `/etc/motd` | base MOTD |
| `blue3/bashrc` | `/root/.bashrc` | root's environment |
| `blue3/bashrc` | `/home/samir/.bashrc` | the `samir` user's environment |
| `blue3/ssh/sshd_config` | `/etc/ssh/sshd_config` | main SSH configuration |
| `blue3/ssh/ipauth.conf` | `/etc/ssh/sshd_config.d/ipauth.conf` | IPs that allow direct access with the root user |
| `blue3/ssh/keyregeneration.conf` | `/etc/ssh/sshd_config.d/keyregeneration.conf` | Lifetime and size (version 1) |
| `blue3/ssh/rhosts.conf` | `/etc/ssh/sshd_config.d/rhosts.conf` | Host permission settings |
| `blue3/ssh/useprivilegeseparation.conf` | `/etc/ssh/sshd_config.d/useprivilegeseparation.conf` | Privilege separation |

Final actions performed:

- Creates the necessary directories in `/target`
- Adjusts the owner of the `samir` user's home
- Enables the `ssh` service
- Tries to mark the `update-motd.d` scripts as executable
- Copies a custom `sources.list` to `/etc/apt/sources.list`

## Dependencies

Packages expected on the build host:

```bash
sudo apt install xorriso isolinux syslinux-utils cpio gzip
```

`md5sum`, `find`, `sed`, `xargs` and `sudo` permission are also used for cleanup and ownership adjustment.

## How to Generate the ISO

1. Place the source Debian ISO in this directory with the name `debian.iso`.
2. Confirm that `blue3/preseed.cfg` and the files in `blue3/` are up to date.
3. Run:

```bash
cd /home/samir/Webs/b3files/www/files.b3.rs/blue3/debian_blue3_iso
sudo bash script-iso.sh
```

Expected output:

- ISO generated at `blue3-debian-YYYYMMDD.iso`
- Log saved to `custom.log`

## Important Notes

- The correct name of the current script is `script-iso.sh`.
- The process's `chown` uses the `BUILD_USER` and `BUILD_GROUP` variables defined at the start of the script.
- The correct preseed boot path is `/cdrom/blue3/preseed.cfg`.
- The `blue3/grub.cfg` file exists in the project, but the current script flow does not copy it directly to the ISO; the boot adjustment is done via `sed` over the extracted ISO.
- The `blue3/blue3.png` file exists in the project, but is not manipulated directly by the current script.
- If the intention is to publish this outside a controlled environment, it is worth reviewing IPs, passwords and SSH rules before pushing to a remote repository.
- The full installation process using a virtual machine with 8 vCPU, 8 GB of RAM and a 60 GB SSD
  - Windows Server 2022 - Hyper-V took 3 minutes and 33 seconds
  - Proxmox 9.1 took 3 minutes and 10 seconds
