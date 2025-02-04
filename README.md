# HomeRunOS
A somewhat secure OS inspired by QubesOS.

Separate different contexts into isolated virtual machines and run apps in those VMs placed on the main desktop.

The rationale for creating HomeRunOS is to have a more relaxed and performative alternative to QubesOS.

We want the option to run graphical applications in dom0 who have direct access to the shared GPU,
and we want things such as watching YouTube from inside VMs not to lag.

In QubesOS running anything in dom0 is a cardinal sin. If dom0 is breached then the whole system is breached, that is why you never run anything in dom0.

In HomeRunOS we place this squarely on the user and ask you to be very mindful what you run in dom0, and always create a separate user to run applications in dom0 to at least have one barrier in place.

HomeRunOS in short:

    - Inspired by QubesOS, but not as secure.
    - Run graphical applications and games (Steam) using a single shared GPU.
    - Requires less hardware to run smoothly.
    - Runs VMs using KVM/QEMU (using virt-manager).
    - Gives more flexibility to the user, and requires more responsibility of the user to not abuse dom0.
    - Run applications in VMs over X11 forwarding or Wayland.
    - Works with XFCE4, GNOME and KDE, but in our experience KDE brings a very smooth experience working with X11/Wayland forwarding,
        also for the VM-to-VM copy to work KDE must be run in dom0, or have `kdialog` available.
    - Copying files between VMs has a more "regular cp" feeling to it but still requires confirmation from a dom0 dialog interacton.
    - HomeRunOS is really just a few shell scripts and can be used from your already existing GNU/Linux OS.

Caveats:
    - Clipboard is not isolated between applications and VMs.
    - There is no dedicated network VM for internet access, instead we do VPN connections from every VM.
    - Networking is by default very open, guest VMs can make network requests to dom0, and vice versa.
    - Management is done over SSH so if clamping down on networking be aware that dom0 need to to SSH requests to the VMs.
    - There is no dedicated USB VM, manage devices in virt-manager UI.

## Install Host / dom0
Either install HomeRunOS on your existing GNU/Linux or install fresh.

If installing fresh we recommend Debian12 with KDE Plasma. XFCE4 works as well, but KDE Plasma has less wrinkles when it comes to remote launcing applications over X11 forwarding or Wayland.

After installation proceed below.

Either make user into sudo or run the commands below as root.

Make user sudo user:  

Create /etc/suoders.d/user as:  
```sh
echo "%$USER ALL = (ALL) NOPASSWD: ALL" >/etc/sudoers.d/$USER
```

### Install virt-manager

```sh
sudo apt install virt-manager libguestfs-tools waypipe
```

Make virt-manager available for regular user:

```sh
echo 'uri_default = "qemu:///system"' >~/.config/libvirt/libvirt.conf

sudo systemctl restart libvirtd.service

sudo usermod -aG libvirt $USER

newgrp libvirt

virsh net-autostart default

virsh net-start default

# test it
virsh list --all
```

### Create SSH key to be used with guest VMs

If no SSH key exists then create one:  

```sh
ssh-keygen -t ed25519
```

### Install HomeRunOS into dom0
```sh
cd
git clone https://github.com/HomeRunOS/homerunos-dom0

Add the `oo` script to the PATH.

For example in `.bashrc` add the following line:
```sh
export PATH="${PATH}:${HOME}/homerunos-dom0"
```

See `oo -h` for help on the oo command which is to manage dom0 with all guest VMs.

## Create a template VM
We recommend using a guest VM with KDE Plasma as it has very little wrinkles, for example Debian 12 with KDE Plasma.

Note that you can also install something like Alpine Linux also. This is a good choice for a small VM running a single application such as a password manager. See appendix section for instructions on Alpine Linux.

Download chosen .iso from debian (or from your chosen distro) and install it using virt-manager.

Install the template VM as this:

    - VM name something as "deb12tpl"
    - username should be "user"
    - no root password so user becomes sudo instead
    - install with SSH server
    - choose KDE Plasma for best experience

After installation configure your template:

    - In KDE Plasma configure the session in the VM to start with Wayland.
    - Disable hibernate and sleep.
    - Make sure SSHD is enabled (systemctl enable ssh)
    - Likely you want to enable X11 forwarding in `/etc/ssh/ssd_config`, set "X11Forwarding yes"
    - Possibly enable TCP forwarding in `/etc/ssh/ssd_config`, set "AllowTcpForwarding yes", this is needed for copying files from another VM.
    - run systemctl restart ssh
    - place the `oocp` file on PATH to be able to copy files from other VMs.


In a shell inside the template VM run the following:

```sh
mkdir ~/.ssh
chmod 700 ~/.ssh
sudo apt update
sudo apt upgrade
apt sudo install waypipe
```

Now leave the VM window and run the following from a dom0 terminal.

First let's see if we can find the template VM:

```sh
oo ip deb12tpl
```

This should yield the local IP of the VM.

If all good then we want to add the dom0 ssh pub key to the template VM.
You also need to accept the pub key of the VM to proceed.

```sh
cat ~/.ssh/id_ed25519.pub | oo ssh deb12tpl tee -a /home/user/.ssh/authorized_keys
```

Now try to ssh into the template without password.

```sh
oo ssh deb12tpl
```

Exit the shell.

Now try to open a windowing program using X11 or Wayland.

If KDE is running in the template VM let's try opening `firefox` and `konsole`
```sh
oo wopen deb12tpl konsole

oo xopen deb12tpl konsole
```

Note that `xopen` for for using X11 Forwarding, and `wopen` is to preferring Wayland with X11 fallback.

### Install HomeRunOS into guest VM
```sh
cd
git clone https://github.com/HomeRunOS/homerunos-guest

Add the guest scripts to the PATH.

For example in `.bashrc` add the following line:
```sh
export PATH="${PATH}:${HOME}/homerunos-guest"
```

### Recommended setup in template VM

Consider:

    - Configure Firefox for duck duck go search and add Privacy Badger as extension.
    - Setup VPN, for example Mullvad. See: https://mullvad.net/en/help/install-mullvad-app-linux
    - Install Mullvad Browser

Configure mullvad-vpn:

```sh
mullvad lan set allow

mullvad auto-connect set on

mullvad account login

mullvad connect

sudo apt install mullvad-browser
```

## Create new VM from template VM
On dom0 run:
```
oo vm-new deb12tpl personal
```

## Appendix

Here follows some optional configurations and tips.

### Create extra account on dom0
We should not use default 'user' account on dom0 for anything but setup.
But to run apps with GPU access we can create another account on dom for this specific purpose.

For example to run Steam games first install Steam as regular user, then run it as the `gfx` user. 

Absolutely do NOT make the `gfx` user a sudo user.

Create user gfx on dom0:  
```sh
sudo adduser gfx
sudo usermod -aG audio gfx
sudo usermod -aG video gfx
```

If gfx user is to have access to USB devices, such as joysticks, then also add to the group `input`.
To have access to raw USB devices also add gfx user to group `dialout`.

Before switching to the gfx user, run:  
```sh
sudo xhost "+local:"
```

Switch to user `gfx`:  
```sh
sudo su gfx1
```

As user `gfx` run application of choice, such as `steam`, then exit shell.

Narrow down xhost again with:  
```sh
sudo xhost "-local:"
```


### Create app launchers on dom0
To be able to launch application of VMs from the desktop create application .desktop files.

Place the app.desktop file in `~/.local/share/applications`.



### Setup Alpine VM
For some specific applications an Alpine Linux VM is the right choice.
Download the .iso and setup the VM in virt-manager.
Follow this guide:  https://linux.how2shout.com/how-to-install-xfce-gui-on-alpine-linux/

Note that virt-sysprep cannot work with Alpine images meaning HomeRunOS cannot clone and prepare Alpine templates,you will have to manage that yourself.


Some notes on making it work:  

Setup with 3 GB disk and 512 MB RAM and 2 vCPUs (keepassxc stalls with only single vCPU).

After starting the ISO in virt-manager:  

```sh
setup-alpine
```

Then on Alpine Guest:  
```sh
vi /etc/ssh/sshd_config # to enable X11Forwarding and AllowTcpForwarding
rc-service sshd restart
adduser user
su user
cd
mkdir .ssh
chmod 700 .ssh
ifconfig # Take note of IP
```

In dom0:  
```sh
cat ~/.ssh/id_ed25519.pub | ssh user@IP tee -a /home/user/.ssh/authorized_keys
```

Back on Alpine as root:  
```sh
setup-xorg-base
apk add xfce4 xfce4-terminal
```

Above is enough to run apps using X11Forwarding.

Also do something as the following if you want to login using the GUI, but that is not required.

```sh
apk add xfce4-screensaver lightdm-gtk-greeter
rc-service dbus start
rc-update add dbus
rc-update add lightdm
rc-service lightdm start
```

### USB
If dom0 user is to have access to USB add user to to both `dialout` and `input` groups.

Attach/detach USB devices from the virt-manager UI.

### Resize images
1. Shutdown VM  
2. Move image to img.qcow2.old  
3. Create new img and expand it

```sh
sudo mv /var/lib/libvirt/images/img.qcow2 img.qcow2.old
sudo virt-filesystems -a img.qcow2 --all --long -h  # Take not of partition to expand
sudo qemu-img create -f qcow2 /var/lib/libvirt/images/img.qcow2 40G
sudo virt-resize --expand /dev/sda1 img.qcow2.old /var/lib/libvirt/images/img.qcow2
```
