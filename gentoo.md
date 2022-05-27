#### Mon installation gentoo

- Boot sur un gentoo iso 
- Préparation des partions avec Parted + lvm + crypsetup [parted](https://wiki.gentoo.org/wiki/Full_Disk_Encryption_From_Scratch_Simplified)
- untar stage3 
- chroot 
- Visualisation du materiel avec => [inxi]("https://github.com/smxi") servira pour compiler le kernel avec ce qu il faut . 
- Interressant voir => https://terminalroot.com/10-fundamental-tips-for-your-gentoo-linux/

##### Schema de partionement : 
```bash
parted -a optimal /dev/nvmxxx
mklabel gpt
unit mb
mkpart primary  1 3 
name 1 grub
set 1 bios_grub on
mkpart primary fat32 3 1024
name 2 boot
set 2 BOOT on 
mkpart primary 1024 -1 
name 3 luks
set 3 lvm on
```


##### Configuration du réseau pour installation via ssh 

**Extraction du stage3**

```bash
tar xpvf stage3-*.tar.bz2 --xattrs-include='*.*' --numeric-owner
```
**Portage repos**
```bash
mkdir /mnt/gentoo/etc/portage/repos.conf
cp /mnt/gentoo/usr/share/portage/config/repos.conf /mnt/gentoo/etc/portage/repos.conf/gentoo.conf 
```
**chroot**
```bash
cp /etc/resolv.conf /mnt/gentoo/etc/resolv.conf 
```
```bash
mount -t proc /proc /mnt/gentoo/proc
mount --rbind /sys /mnt/gentoo/sys
mount --make-rslave /mnt/gentoo/sys
mount --rbind /dev /mnt/gentoo/dev
mount --make-rslave /mnt/gentoo/dev 
```
```bash
test -L /dev/shm && rm /dev/shm && mkdir /dev/shm
mount -t tmpfs -o nosuid,nodev,noexec shm /dev/shm
chmod 1777 /dev/shm 
```

```bash
chroot /mnt/gentoo /bin/bash
source /etc/profile 
```

```bash
mount /dev/sda2 /boot
```
***history***
```bash
eselect profile list
emerge-webrsync 
eselect  profile list
eselect  profile set 6 
echo Europe/Paris > /etc/timezone
emerge --config sys-libs/timezone-data
```

### Mise a jours
```bash
emerge -avuDN @world
```

##### Déclaration des locales
```bash
nano -w /etc/locale.gen
fr_FR.UTF-8 UTF-8
locale-gen 
eselect  locale list
eselect  locale set 4 
. /etc/profile
export PS1="(chroot)$PS1"
   ```
#### Clavier
 ```bash
nano -w /etc/conf.d/keymaps 
env-update && source /etc/profile && export PS1="[chroot] $PS1"
```
   
#### Synchro heure systeme et materiel
```bash
nano -w /etc/conf.d/hwclock
hwclock --systohc
```
##### Declaration des point de montages .
```bash
nano -w /etc/fstab 
blkid >> /etc/fstab 
nano -w /etc/fstab 
```
###### Installation des source du Kernel et le outils pour le compiler (genkernel) +  outils pratiques (lspci ) 
   
 **important avant d isntaller les firmware**
   
   dans  /etc/portage/make.conf : 
   
```bash
ACCEPT_LICENSE='*'
```
```bash
emerge sys-kernel/gentoo-sources 
emerge sys-kernel/genkernel
emerge sys-fs/cryptsetup
emerge -a pciutils usbutils
   
env-update && source /etc/profile && export PS1="[chroot] $PS1"

echo"sys-kernel/linux-firmware linux-fw-redistributable no-source-code" >> /etc/portage/package.license
emerge  -a linux-firmware
eselect  profile list
```
##### Compilation du kernel un peu moins de 2h sur un : Intel(R) Core(TM) i5-8365U CPU @ 1.60GHz
```bash
cd /usr/src/linux
make menuconfig
```
```bash
make && make modules_install && make install
```
ou via genkernel : 
```bash
ls /usr/src/linux-5.10.27-gentoo/
vi /etc/genkernel.conf 
nano  -w /etc/genkernel.conf 
 
ln -s /usr/src/linux-5.10.27-gentoo/ /usr/src/linux 
genkernel all --luks --lvm --no-zfs 
 
 - Pour générer uniquement l initramfs : 
genkernel --luks --lvm initramfs 
nano -w /etc/conf.d/hostname 
useradd -m -G users,wheel,audio,cdrom,video,portage -s /bin/bash  seb
   ```
##### Installation grub + Hook sur le grub pour crypsetup 
```bash
echo "sys-boot/grub:2 device-mapper" >> /etc/portage/package.use/sys-boot 
emerge -av grub
mount /boot/
emerge -a vim 
grub-install --target=x86_64-efi --efi-directory=/boot 
vi /etc/default/grub 
```
```bash
GRUB_CMDLINE_LINUX="net.ifnames=0 dolvm crypt_root=UUID=7c488d8a-3afc-4dec-838f-304c65776e86  root_trim=yes"
GRUB_THEME="/boot/grub/themes/Atomic-GRUB2-Theme/Atomic/theme.txt"
```
    - dolvm 
    - crypt_root (partitionon lvm )
    - net.ifnames (appelation des interfaces réseau , ethx..)
```bash
grub-mkconfig -o /boot/grub/grub.cfg 
 ```
###### Services de bases .
 ```bash
emerge -a rsyslog logrotate
rc-update addrsyslog default 
rc-update add rsyslog default 
emerge -a vixie-cron
emerge -a ntp
rc-update add ntpd default
emerge -a gentoolkit portage-utils htop app-misc/screen eix nmon
vim /etc/conf.d/net
vim /etc/conf.d/net-online 
emerge -a dhcpcd 
 ```
##### Instalation de gnome-light
  dans /etc/portage/make.conf
  #Ajout spec
  
 ```bash
USE="bindist python X gtk gnome elogind static-libs"
MAKEOPTS="-j9"
#MAKEOPTS="-j2"
#LINGUAS="fr" linguas est visiblement déprécié 
L10N="fr"
VIDEO_CARDS="intel i915 fbdev vesa"
EMERGE_DEFAULT_OPTS="${EMERGE_DEFAULTS_OPTS} --quiet-build=y"
INPUT_DEVICES="libinput evdev synaptics keyboard mouse"
### variable mirroir permet l accélération des ebuild .
GENTOO_MIRRORS="https://gentoo.c3sl.ufpr.br/ http://gentoo.c3sl.ufpr.br/ rsync://gentoo.c3sl.ufpr.br/gentoo/
```

**Update /etc/portage/make.conf**
```bash
####Seb
CPU_FLAGS_X86="aes avx avx2 f16c fma3 mmx mmxext pclmul popcnt rdrand sha sse sse2 sse3 sse4_1 sse4_2 sse4a ssse3"
#USE="bindist python X gtk gnome elogind static-libs"
USE="bindist alsa python X xcb xkb elogind dbus udev ffmpeg rust-bin bluetooth\
     -wayland -systemd -rust -pulseaudio"
MAKEOPTS="-j9"
#MAKEOPTS="-j2"
#LINGUAS="fr" linguas est visiblement déprécié
L10N="fr"
#VIDEO_CARDS="intel i915 fbdev vesa"
VIDEO_CARDS="amdgpu radeonsi"
EMERGE_DEFAULT_OPTS="${EMERGE_DEFAULTS_OPTS} --quiet-build=y"
INPUT_DEVICES="libinput evdev synaptics keyboard mouse"
### variable mirroir permet l accélération des ebuild .
GENTOO_MIRRORS="https://gentoo.c3sl.ufpr.br/ http://gentoo.c3sl.ufpr.br/ rsync://gentoo.c3sl.ufpr.br/gentoo/"
GRUB_PLATFORM="efi-64"
### qemu/kvm
QEMU_SOFTMMU_TARGETS="x86_64"
QEMU_USER_TARGETS="x86_64"
ACCEPT_KEYWORDS="~amd64"
```
pour pouvoir compiler net-libs/webkit-gtk il faut supprimer ***bindist*** de **USE**
ou compiler séparément . 

#### Install env graphique:
```bash
emerge --ask x11-wm/i3 x11-misc/dmenu x11-misc/i3lock xinit xorg
rc-update add elogind boot
```

xorg driver :

```bash
emerge --ask --verbose --oneshot x11-base/xorg-drivers
emerge -a sys-power/acpi
/etc/conf.d/display-manager
DISPLAYMANAGER="gdm"

emerge -a gnome-base/gnome-light  ### Trés long..
rc-update add dbus default
rc-update add display-manager default 
rc-update add openrc-settingsd default 
rc-update  add elogind boot 
gpasswd -a seb video 
openrc
emerge -a setxkbmap
```
Certain packages ont besoin de connaitre l architecture sinon ils seront masqué , exemple : sys-apps/bat . 
Pour remédier il faut ajouter le bon keyword a portage : 
```bash
echo 'ACCEPT_KEYWORDS="~amd64"' | sudo tee -a /etc/portage/make.conf
```


#### clavier X11
  cat /etc/X11/xorg.conf.d/00-keyboard.conf 
```bash
Section "InputClass"
Identifier "system-keyboard"
MatchIsKeyboard "on"
Option "XkbLayout" "fr,us"
Option "XkbVariant" ",,"
EndSection
```
#### Theme gnome
- theme = Nordic
```bash
- mkdir -p ~/.themes ( unzip du theme dans ce repertoire )
- emerge gnome-tweaks ( configuration du theme ...)
```
##### Fonts

```bash
sudo emerge -av media-fonts/noto media-fonts/noto-cjk
sudo eselect fontconfig enable 66-noto-mono.conf
sudo eselect fontconfig enable 70-noto-cjk.conf
```


#### Alsa sound
#installation de alsa 
facultatif : 
```bash
euse -E alsa
emerge --ask --changed-use --deep @world
```
obligatoire : 
```bash
emerge --ask media-sound/alsa-utils
```
**Gestion luminosité i3wm**
```bash
emerge dev-libs/light
```
**Création regle udev**
/etc/udev/rules.d/90-backlight.rules
```bash
ACTION=="add", SUBSYSTEM=="backlight", RUN+="/bin/chgrp video /sys/class/backlight/%k/brightness"
ACTION=="add", SUBSYSTEM=="backlight", RUN+="/bin/chmod g+w /sys/class/backlight/%k/brightness"
ACTION=="add", SUBSYSTEM=="leds", RUN+="/bin/chgrp video /sys/class/leds/%k/brightness"
ACTION=="add", SUBSYSTEM=="leds", RUN+="/bin/chmod g+w /sys/class/leds/%k/brightness"
```
voir => https://github.com/haikarainen/light#permissions

----
**bluetooth**
```bash
/etc/init.d/bluetooth start
bluetoothctl 
power on
agent on 
scan on

#Premiere connection
pair 11:BB:CC:FF:..
connect 11:BB:CC:FF:..

scan off
exit
```
----

alsa ~/.asoundrc
```bash
#pcm.plugdmix {
#    type plug
#    slave.pcm "dmix"
#}
#
#pcm.!default {
#    type hw
#    card 1
#}
#
#ctl.!default {
#    type hw
#    card 1
#}
#
pcm.dmixed {
    type asym
    playback.pcm {
        type dmix
        ipc_key 5678293
        ipc_perm 0660
        ipc_gid audio

        slave {
            channels 2 # Make 6 or 5.1 channel
            pcm {
                format S16_LE # S32_LE
                rate 48000 # Can also be 44100
                type hw
                card 0 # Your card
                device 0 # Your device
                subdevice 0 # Important?
            }

            period_size 1024
            buffer_size 8192
        }

        bindings {
            0 0
            1 1
# Uncomment below if using 6 channel
#           2 2
#           3 3
#           4 4
#           5 5
        }
    }
    capture.pcm "hw:0"
}

pcm.!default {
    type plug
    slave.pcm "dmixed"
}

#
pcm.!default {
    type hw
    card 2
}

ctl.!default {
    type hw
    card 2
}

```
