# Install Pop!_OS 22.04 on Lenovo T14-G3-AMD (2022) with full disk encryption and ZFS rootfs

```
System Information
        Manufacturer: LENOVO
        Product Name: 21CF004PGE
        Version: ThinkPad T14 Gen 3
        SKU Number: LENOVO_MT_21CF_BU_Think_FM_ThinkPad T14 Gen 3
        Family: ThinkPad T14 Gen 3        
```

For now, no specifics to T14-G3-AMD, so follow my [Root Boot Encrypted ZFS Guide](https://gist.github.com/longsleep/48a585219e1c7b5d04b827ed02b029de).

The following guide assumes that you have completed installation and are booted
into the installed system. 

## After first boot

Open up a terminal and become root (`sudo -i`)

```
zsysctl save -s after_install
```

```
vi /etc/gdm/custom.conf
# Comment WaylandEnable=false.
# Save and quit.
```

Now `systemctl restart gdm3` and you will be signed out. Now you can choose 
Wayland session on the bottom right (after selecting your user).

Also install some basics:

```
apt install --yes \
    gdebi \
    tlp \
    fonts-powerline \
    tmux \
    zsh \
    solaar \
    g810-led \
    alacritty \
    python3-pip \
    lm-sensors
```

This pretty much gives me the basic software to use my [bin-scripts](https://github.com/longsleep/bin-scripts) repository.
Its now left to the reader how to configure the individual environment. 


## Issues

The Hardware platform of the T14 Gen3 AMD is pretty new, so there are some Linux compatibilitiy issues.

I see the following issues (Kernel 5.19.0-76051900-generic #202207312230~1660780566~22.04~9d60db1):

- The internal microphone does not work (or very low volume)
- My external USB microphone is way to low volume (not usable) -> seems fixed, see below
- Suspend does not work (no matter if S3 Linux or Windows and Linux is set in BIOS), both "s2idle" and "deep" crashes, hang or do not properly resume in various variants
- Hotkeys to switch workspaces do not work, especially when using Workspace matrix -> fixed, see below
- Fan speed indicator sometimes does show bogus values when fan is off -> not really a problem, see below

### Fixing internal microphone

Others reported problems with the microphone for older T14 generations, but 
Kernel 5.19 seems to have the quirks for the acp6x in place for `21CF` - so it
seems to be something else (see https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/sound/soc/amd/yc/acp6x-mach.c?h=v5.19#n80)

Not sure if Kernel 6.0 would improve things, but but at the time of writing ZFS
for linux does not support Kernel 6.0 - there is a patch set (https://github.com/openzfs/zfs/pull/13886)
which includes some fixes for 6.0 compat.

We might be able to backport via DKMS (https://www.collabora.com/news-and-blog/blog/2021/05/05/quick-hack-patching-kernel-module-using-dkms/).

TODO

### Fixing external USB microphone

Not sure about this one, the problem went away after using alsamixing and sound settings volume control a couple of times.

This problem is now fixed.

### Fixing suspend

TODO

### Fix hotkeys to switch workspaces

Useful with the following extension: https://github.com/mzur/gnome-shell-wsmatrix

```
gsettings set org.gnome.desktop.wm.keybindings switch-to-workspace-left '["<Control><Alt>Left"]'
gsettings set org.gnome.desktop.wm.keybindings switch-to-workspace-right '["<Control><Alt>Right"]'
gsettings set org.gnome.desktop.wm.keybindings switch-to-workspace-up '["<Control><Alt>Up"]'
gsettings set org.gnome.desktop.wm.keybindings switch-to-workspace-down '["<Control><Alt>Down"]'
gsettings set org.gnome.desktop.wm.keybindings move-to-workspace-left '["<Shift><Control><Alt>Left"]'
gsettings set org.gnome.desktop.wm.keybindings move-to-workspace-right '["<Shift><Control><Alt>Right"]'
gsettings set org.gnome.desktop.wm.keybindings move-to-workspace-up '["<Shift><Control><Alt>Up"]'
gsettings set org.gnome.desktop.wm.keybindings move-to-workspace-down '["<Shift><Control><Alt>Down"]'
```

This problem is now fixed.

## Fix fan speed indicator

Not a big deal, but the fan speed sometimes get reported wrong. It shows 65535 
in the vitals Gnome 3 extension while the fan really is off.

It seems that there is a second fan sensor which reports this bogus value.

```
sensors|grep fan
fan1:           0 RPM
fan2:        65535 RPM
```

TODO




## Secure boot for DKMS Kernel modules

NOTE: This section does not work, so ignore for now.

Since we use Pop!_OS, there might be additional DKMS Kernel modules which
need to be singed. A Machine Owner Key (MOK), needs to be created and enrolled
to Secure Boot. That key is then used to sign all extra Kernel modules.

All the next steps assume you are in a `root` shell.

#### 1. Generate new key and certificate

```
openssl req -new -x509 \
    -newkey rsa:2048 -keyout /root/dkms.key \
    -outform DER -out /root/dkms.der \
    -nodes -days 7300 -subj "/CN=DKMS Signing MOK 2022"
```

#### 2. Enroll the public key

```
mokutil --import /root/dkms.der
```

A password is required to be set, it is needed later again.

Now reboot the computer.

```
reboot
```

You will see the MOK interface. Press any key to enter it. And select 
"Enroll MOK", verify the key, then select "Continue". You will need the password
which you entered earlier. Click "OK" and then reboot into the system again.

The key is now available in `/proc/keys`.

#### 3. Configure DKMS to sign built modules automatically with the MOK

```
vi /etc/dkms/framework.conf
# Uncomment the # sing_tool line at the bottom.
# Save and quit.

```

Well, that did not work - figure out a way.



