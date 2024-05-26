# Autoinstalling Ubuntu

This is a guide for using Ubuntu's autoinstall feature for rapid installation of Ubuntu primarily on bare metal machines

## Needed tools

To get started we'll need three tools, 7zip, wget and xorriso:

```
#apt install -y 7zip wget xorriso
```

## Getting a Ubuntu image to work with

```
$mkdir u24.04-autoinstall-ISO
$cd u24.04-autoinstall-ISO
$mkdir source-files
$wget http://mirror.csclub.uwaterloo.ca/ubuntu-releases/noble/ubuntu-24.04-live-server-amd64.iso
```

## Extracting the ISO

```
$7z -y x jammy-live-server-amd64.iso -oubuntu-24.04-live-server-amd64.iso
$cd source-files/
$mv  '[BOOT]' ../BOOT
```

Take note! There is no space after the `-o` in the 7z command.

## Modifying GRUB

```
$nano boot/grub/grub.cfg
```

Add the following menu entry above the existing menu entries:

```
menuentry "Autoinstall Ubuntu Server" {
    set gfxpayload=keep
    linux   /casper/vmlinuz quiet autoinstall ds=nocloud\;s=/cdrom/server/  ---
    initrd  /casper/initrd
}
```

## Creating the autoinstall directives

```
$mkdir server
$touch server/meta-data
```

This `server` folder is the one referenced in the menu entry above so you can have multiple entries with slightly different manifests. The `meta-data` file is intentionally left blank.

Next we will populate a `user-data` file, this is the meat and potatoes of the whole thing. The official documentation for the user-data file can be found here: https://canonical-subiquity.readthedocs-hosted.com/en/latest/reference/autoinstall-reference.html I have included a working example below:

```yaml
#cloud-config
autoinstall:
  # version is an Autoinstall required field.
  version: 1

  # This adds the default ubuntu-desktop packages to the system.
  # Any desired additional packages may also be listed here.
  packages:
    - ubuntu-server

  # User creation can occur in one of 3 ways:
  # 1. Create a user using this `identity` section.
  # 2. Create users as documented in cloud-init inside the user-data section,
  #    which means this single-user identity section may be removed.
  # 3. Prompt for user configuration on first boot.  Remove this identity
  #    section and see the "Installation without a default user" section.
  identity:
    realname: 'UntouchedWagons'
    username: untouchedwagons
    # A password hash is needed. `openssl passwd -6 $CLEARTEXT_PASSWORD` can help.
    password: ''
    hostname: ubuntu-test

  locale: en_US.UTF-8
  keyboard:
    layout: us

  package_update: true
  package_upgrade: true

  # Subiquity will, by default, configure a partition layout using LVM.
  # The 'direct' layout method shown here will produce a non-LVM result.
  storage:
    swap:
      size: 0
    layout:
      name: direct

  ssh:
    allow-pw: true
    install-server: true
    authorized-keys:
      - ssh-key 1
      - ssh-key 2

  network:
    network:
      version: 2
      ethernets:
        enp6s18:
          dhcp4: true
          dhcp-identifier: mac

  late-commands:
    - curtin in-target -- update-grub
    - curtin in-target -- apt-get install -y cloud-init
    - curtin in-target -- apt-get autoremove -y
```

Note that due to the use of predictable interface naming, it can be difficult to predict what the name of the interface will be. In a Proxmox VM, Ubuntu tends to name the virtio NIC `enp6s18`, your mileage may vary; it could be `eth0`, `eno1` or something completely different.

## Rebuilding the ISO

I lifted the command for rebuilding the ISO nearly completely from the Puget Systems link listed below, I have only a vague idea of what each part does.

```
$xorriso -as mkisofs -r \
  -V 'Ubuntu 24.04 LTS AUTO (EFIBIOS)' \
  -o ../ubuntu-24.04-autoinstall.iso \
  --grub2-mbr ../BOOT/1-Boot-NoEmul.img \
  -partition_offset 16 \
  --mbr-force-bootable \
  -append_partition 2 28732ac11ff8d211ba4b00a0c93ec93b ../BOOT/2-Boot-NoEmul.img \
  -appended_part_as_gpt \
  -iso_mbr_part_type a2a0d0ebe5b9334487c068b6b72699c7 \
  -c '/boot.catalog' \
  -b '/boot/grub/i386-pc/eltorito.img' \
    -no-emul-boot -boot-load-size 4 -boot-info-table --grub2-boot-info \
  -eltorito-alt-boot \
  -e '--interval:appended_partition_2:::' \
  -no-emul-boot \
  .
```

If everything went successfully you should see the message `Writing to 'stdio:../ubuntu-24.04-autoinstall.iso' completed successfully.`

And that's it. You can upload the ISO to your hypervisor of voice, write it to a USB drive using RUFUS or copy it to your IODD external virtual ODD.

## Things of note

If you intend to use this for a VM on a Proxmox host I recommend installing `qemu-guest-agent` by adding the following line to the `late-commands` subsection:

```
- curtin in-target -- apt-get install -y qemu-guest-agent
```

Qemu Guest Agent is really useful for seeing information like what NICs are in use in the VM, their IP addresses and et cetera.

## Sources

* https://www.pugetsystems.com/labs/hpc/ubuntu-22-04-server-autoinstall-iso/
* https://askubuntu.com/questions/1440007/qemu-guest-agent-fails-at-autoinstall
