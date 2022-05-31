# nix-bitcoin Setup

## 0. Concepts

### Motivations

### Terminology

* Nix - a language and build system
* NixOS - a Linux distribution
* nix-bitcoin - packages and modules which turn a `NixOS` host into a bitcoin node

### Strategy

Setting up a `nix-bitcoin` node is divided into two phases:

1. Preparing host machine and installing the `NixOS` Linux distribution to it
2. Creating a `nix` deployment environment on a different machine, and then deploying the `nix-bitcoin` packages and modules to the `NixOS` host machine. Going forward, this machine will act as your terminal from which you will administrate your bitcoin node.

## 1. Preparation

### Generate ssh key

On a separate machine:

```
$ ssh-keygen -t ed25519
$ cat ~/.ssh/id_ed25519.pub
```
Save the generated private and public keys for use in the nix configuration and the remote deployment machine

### Create the NixOS install media

Download the image from `https://nixos.org/download.html#nixos-iso`

Burn it to a USB drive or disc: `sudo dd if=nix.iso of=/dev/rdiskN`

### Boot the media

Edit your BIOS UEFI / Boot order settings accordingly

## 2. NixOS Server Setup

### Disk Partitioning

Depending on number of drives, performance needs, and risk tolerance for failure scenarios, other setups may vary wildly. You can pool multiple disks together with no redundancy, `mirror` for replication, or use parity options like `raidz1`/`raidz2`/`raidz3`.

This setup will make use of 6 disks in a `raidz2` configuration.

Set up environment variables
```bash
$ $DISK1=/dev/disk/by-id/ata-WDC_WD40EFZX-68AWUN0_WD-WX12DUUUUUUU
$ $DISK2=/dev/disk/by-id/ata-WDC_WD40EFZX-68AWUN0_WD-WX12DVVVVVVV
$ $DISK3=/dev/disk/by-id/ata-WDC_WD40EFZX-68AWUN0_WD-WX12DWWWWWWW
$ $DISK4=/dev/disk/by-id/ata-WDC_WD40EFZX-68AWUN0_WD-WX12DXXXXXXX
$ $DISK5=/dev/disk/by-id/ata-WDC_WD40EFZX-68AWUN0_WD-WX12DYYYYYYY
$ $DISK6=/dev/disk/by-id/ata-WDC_WD40EFZX-68AWUN0_WD-WX12DZZZZZZZ
```

Create the `EFI` boot partition
```bash
$ sgdisk -n3:1M:+512M -t3:EF00 $DISK1
```

Create the OS partition
```bash
$ sgdisk -n1:0:0 -t1:BF01 $DISK1
```

Copy the partition table across all devices
```bash
$ sfdisk --dump $DISK1 | sfdisk $DISK2
$ sfdisk --dump $DISK1 | sfdisk $DISK3
$ sfdisk --dump $DISK1 | sfdisk $DISK4
$ sfdisk --dump $DISK1 | sfdisk $DISK5
$ sfdisk --dump $DISK1 | sfdisk $DISK6
```

Format the boot partitions:
```bash
$ mkfs.vfat $DISK1-part3
$ mkfs.vfat $DISK2-part3
$ mkfs.vfat $DISK3-part3
$ mkfs.vfat $DISK4-part3
$ mkfs.vfat $DISK5-part3
$ mkfs.vfat $DISK6-part3
```

### ZFS raidz2 Pool creation

```bash
$ zpool create -o ashift=12 -O atime=off -O encryption=aes-256-gcm \
             -O acltype=posixacl -O keyformat=passphrase -O xattr=sa \
             -O mountpoint=none -O compression=lz4 \
             zroot raidz2 \
             $DISK1-part1 $DISK2-part1 $DISK3-part1 \
             $DISK4-part1 $DISK5-part1 $DISK6-part1
```
_(At this point, you will be prompted for a disk encryption passphrase)_

Create the `zfs` filesystems:
```bash
$ zfs create -o mountpoint=none zroot/root
$ zfs create -o mountpoint=legacy zroot/root/nixos
$ zfs create -o mountpoint=legacy zroot/var
$ zfs create -o mountpoint=legacy zroot/home
```

### Mount install targets

Mount the `Nixos` root and create child mountpoints:
```bash
$ mount -t zfs zroot/root/nixos /mnt

$ mkdir /mnt/home
$ mkdir /mnt/var
$ mkdir /mnt/boot
```

Mount the child filesystem mountpoints and `EFI` boot partition:
```bash
$ mount -t zfs zroot/home /mnt/home
$ mount -t zfs zroot/var /mnt/var
$ mount /mnt/boot $DISK1-part3
```

### Generate config:
```bash
$ nixos-generate-config --root /mnt
```


### Retrieve host id:
Get the __HOST ID__ of the machine, save it for later:
```bash
$ head -c 8 /etc/machine-id
```

### Customize the `configuration.nix` file:
Note that `openssh.authorizedKeys.keys` item value is the contents of your ssh public key file (`id_ed25519.pub`)
```javascript
boot.loader.systemd-boot.enable = true;
boot.loader.efi.canTouchEfiVariables = true;
boot.supportedFilesystems = [ "zfs" ];
boot.zfs.requestEncryptionCredentials = true;

networking.hostName = "nixnode";
networking.hostId = "[HOST ID GOES HERE]";

time.timeZone = "UTC";

users.users.root = {
	openssh.authorizedKeys.keys=[
		"ssh-ed25519 AAAA... user@nix-bitcoin-deploy"
	];
}

environment.systemPackages = with pkgs; [
	vim
	wget
	curl
	git
];

services.openssh.enable = true;
services.openssh.permitRootLogin="yes";

services.zfs.autoSnapshot.enable = true;
services.zfs.autoScrub.enable = true;
```

### Install

```bash
$ nixos-install
```
_(You will be prompted to create the root password)_

### Get the host's IP address:
```bash
$ ifconfig
```

## 3. Nix Deployment Environment

### Create a Debian VM / Container / Box

```bash
$ sudo apt install curl git gnupg2 dirmgr
```

### Retrieve nix install script, signature, and public signing key
```bash
$ curl -o install-nix-2.7.0 https://releases.nixos.org/nix/nix-2.7.0/install
$ curl -o install-nix-2.7.0.asc https://releases.nixos.org/nix/nix-2.7.0/install.asc
```

### Verify install script

```bash
$ gpg2 --keyserver hkps://keyserver.ubuntu.com --recv-keys B541D55301270E0BCF15CA5D8170B4726D7198DE
$ gpg2 --verify ./install-nix-2.7.0.asc
```

### Run install script

```bash
$ sh ./install-nix-2.7.0 --daemon
```

### Setup Deployment Directory

```bash
$ git clone https://github.com/fort-nix/nix-bitcoin
```

```bash
$ cd nix-bitcoin/examples
$ nix-shell
```

```bash
$ fetch-release > nix-bitcoin-release.nix
```

```bash
$ cd ../../
$ mkdir nix-bitcoin-node
$ cd nix-bitcoin-node
$ cp -r ../nix-bitcoin/examples/{nix-bitcoin-release.nix,configuration.nix,shell.nix,krops,.gitignore} .
```

### Configure for remote deployment to NixOS machine

Copy the `id_ed25519` ssh private key to `~/.ssh`

Change the permissions of the private key:
```bash
$ chmod go-rwx ~/.ssh/id_ed25519
```

in `~/.ssh/config`:
```bash
Host bitcoin-node
        Hostname                [NIXOS HOST IP ADDRESS GOES HERE]
        IdentityFile            ~/.ssh/id_ed25519
        User                    operator
        PubkeyAuthentication    yes
        AddKeysToAgent          yes
```

Edit `nix-bitcoin-node/krops/deploy.nix`:
```javascript
target = "root@bitcoin-node";
```

Retrieve remote NixOS hardware-configuration.nix and configuration.nix:

```bash
$ scp bitcoin-node:/etc/nixos/hardware-configuration.nix hardware-configuration.nix
$ scp bitcoin-node:/etc/nixos/configuration.nix remote-configuration.nix
```

Edit the nix-bitcoin-node/configuration.nix

```javascript
{ config, pkgs, lib, ... }: {
  imports = [
    <nix-bitcoin/modules/presets/secure-node.nix>
    <nix-bitcoin/modules/presets/hardened.nix>
    ./hardware-configuration.nix
    ./remote-configuration.nix
  ];


  ### CLIGHTNING
  services.clightning.enable = true;
  services.clightning.plugins.clboss.enable = true;
  services.clightning.plugins.summary.enable = true;
  services.clightning.extraConfig = ''
    alias=my_nix_node
  '';

  ### RIDE THE LIGHTNING
  services.rtl.enable = true;
  services.rtl.nodes.clightning = true;

  ### ELECTRS
  services.electrs.enable = true;

  nix-bitcoin.configVersion = "0.0.65";
}
```

Enter deployment environment:
```bash
$ nix-shell
$ deploy
```
Following deployment, you will have a bitcoin node. Do not be surprised if resource use is intensive for hours or perhaps a couple days. `bitcoind` will perform its download and validation of the entire blockchain (less than half a terabyte, given pruning wasn't enabled), and `c-lightning` will also take some time to bootstrap following `bitcoind`'s initial block download (IBD).

To access your node:
```
ssh bitcoin-node
```

# LFG ┗(°0°)┛

## Notes:

### Login to Ride The Lightning (RTL)

By default, the node will not allow connections over the local network or the internet (tor is allowed).  If you don't want to change this, you can still tunnel traffic to RTL via your SSH connection.

Alter your `~/.ssh/config` to add port forwarding to the bitcoin-node `Host` configuration:

```
Host bitcoin-node
        Hostname		[NIXOS HOST IP ADDRESS GOES HERE]
	...
	LocalForward 		3000 localhost:3000
```

Once you've configured forwarding, connect to your node from your deployment machine:

`ssh bitcoin-node`

While you're in an SSH terminal session, grab the __RTL password__:

`cat /var/src/secrets/rtl-password`

on the deployment machine, open a browser and navigate to `http://localhost:3000`.

Use the __RTL password__ to log in

### Connect Zeus over local network:

#### Assumptions:
1. Your local subnet is `192.168.1.1` through `192.168.1.255`
2. You have enabled Core Lightning (`clightning`) and Ride the Lightning (`rtl`) in your deployment `configuration.nix`:
	```
	services.clightning.enable = true;
	services.rtl.enable = true;
	services.rtl.nodes.clightning = true;
	```

In your deployment `configuration.nix` file, either add or edit the networking.firewall.extraCommands settings:

```
networking.firewall.extraCommands = ''
    iptables -A nixos-fw -p tcp --source 192.168.1.0/24 --dport 3001 -j nixos-fw-accept
    iptables -A nixos-fw -p udp --source 192.168.1.0/24 --dport 3001 -j nixos-fw-accept
    iptables -A nixos-fw -p tcp --source 192.168.1.0/24 --dport 4001 -j nixos-fw-accept
    iptables -A nixos-fw -p udp --source 192.168.1.0/24 --dport 4001 -j nixos-fw-accept
'';
```

and widen up the `cl-rest` service's allowed IP addresses. :

```
 systemd.services.cl-rest.serviceConfig.IPAddressAllow = lib.mkForce "127.0.0.1/32 ::1/128 169.254.0.0/16 192.168.1.0/24";

```

### Change your lightning node alias

In your deployment environment's `configuration.nix` file, either add or edit the clightning `extraConfig` setting:

```
  services.clightning.extraConfig = ''
    alias=bad_for_education
  '';
```
