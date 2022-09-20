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
    lm-sensors \
    xsel
```

This pretty much gives me the basic software to use my [bin-scripts](https://github.com/longsleep/bin-scripts) repository.
Its now left to the reader how to configure the individual environment.


## Issues

The Hardware platform of the T14 Gen3 AMD is pretty new, so there are some Linux compatibilitiy issues.

I see the following issues (Kernel `5.19.0-76051900-generic #202207312230~1660780566~22.04~9d60db1`):

- The internal microphone volume is very low volume and unusable
- ~My external USB microphone is way to low volume (not usable)~ -> **seems fixed, see below**
- ~Suspend does not work with Wifi is connected (no matter if S3 Linux or Windows and Linux is set in BIOS), both "s2idle" and "deep" crashes, hang or do not properly resume in various variants~ -> **workaround has been made, see below**
- ~Hotkeys to switch workspaces do not work, especially when using Workspace matrix~ -> **fixed, see below**
- Fan speed indicator sometimes does show bogus values when fan is off -> **not really a problem, see below**


### Fixing internal microphone volume

Others reported problems with the microphone for older T14 generations, but
Kernel 5.19 seems to have the quirks for the acp6x in place for `21CF` - so it
seems to be something else (see https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/sound/soc/amd/yc/acp6x-mach.c?h=v5.19#n80)

Not sure if Kernel 6.0 would improve things, but but at the time of writing ZFS
for linux does not support Kernel 6.0 - there is a patch set (https://github.com/openzfs/zfs/pull/13886)
which includes some fixes for 6.0 compat.

Backporting ZFS to support Kernel 6.0 (see below in the fixing suspend section)
did unfortunately not fix the microphone. It seems that its volume is way too 
low and is barely audible even at max level (also set via alsamixer).

Others have reported [similar issues](https://bbs.archlinux.org/viewtopic.php?pid=2056452) wit the `acp6x` based sound card.

Pop!_OS uses Wireplumber, thus to configure alsa settings we must add a 
corresponding Wireplumber configuration rule like so.

```
mkdir -p /etc/wireplumber/main.lua.d
cat <<'EOF' > /etc/wireplumber/main.lua.d/51-alsa-fix-t14g3-mic.lua
rule = {
  matches = {
    {
      { "device.name", "equals", "alsa_card.pci-0000_04_00.6" },
    },
  },
  apply_properties = {
    -- Disable UCM profile to fix microphone.
    ["api.alsa.use-ucm"] = false,
  },
}

table.insert(alsa_monitor.rules, rule)
EOF

systemctl --user restart wireplumber

Sadly, this did not help and looses the sink entirely.

TODO, remains unfixed.


### Fixing external USB microphone

Not sure about this one, the problem went away after using alsamixing and sound settings volume control a couple of times.

This problem is now fixed.


### Fixing suspend

By default neither of the sleep modes possible in BIOS worked for me. One part
of the problem seem to be NetworkManager which simply hangs on suspend and makes
things fail completely. So for any successful sleep, NetworkManager needs to be
stopped. Then it can sleep successfully for both supported mem_sleep modes
(`s2idle`, deep), but only wakes up again for `s2idle`.

So first task is to get the latest Linux Kernel which at the time of writing is
6.0 (release candidate still). It can be installed easily thanks to the [Ubuntu Mainline Kernel PPA](https://kernel.ubuntu.com/~kernel-ppa/mainline/) - but
there is a problem - like described in the microphone issue above, at the time
of writing ZFS for linux does not support Kernel 6.0. So i backported the few
relevant compatibility fixes from [here](https://github.com/openzfs/zfs/pull/13886) and
rebuilt an updated zfs-onlinux package (find the packaging [here](https://github.com/longsleep/zfs-linux-deb).
That makes Kernel 6.0 run quite nicely, and for this is a requirement for the
suspend workaround described below.

Waking up from `s2idle` requires the latest firmware to be insalled. I fetched
`linux-firmware-20220913` which is the newest at the time of writing and it
recently got updates for the amdgpu files.

Extract the latest linux-firmware into `/lib/firmware/updates` and update the
inird with `update-initramfs -c -k all`. Then reboot.

Now to NetworkManager. For now we simply add two systemd services to stop
NetworkManager before sleep and to restart it after resume.

```
cat <<'EOF' > /etc/NetworkManager/dispatcher.d/pre-down.d/99-t14-g3-amd-suspend-fix
#!/bin/sh -ex

# This is a hack to workaround hang if the T114 G3 AMD wifi device. This does
# the trick until a proper fix is found by simply disabling Wifi completely if
# the main wifi is about to go down. It has to be in the pre-down stage as the
# down stage is too lage and hangs for unknown reason.

if [ "$2" = "connectivity-change" ]; then
    exit 0;
fi

if [ -z "$1" ]; then
    echo "$0: called with no interface" 1>&2
    exit 1;
fi

if [ "$1" != "wlp2s0" ]; then
    exit 0;
fi

case $2 in
   pre-down)
       export
       if [ "$(nmcli -f STATE -t g status 2>/dev/null)" = "asleep" ]; then
           rfkill block wifi
       fi
       ;;

esac
EOF
chmod 755 /etc/NetworkManager/dispatcher.d/pre-down.d/99-t14-g3-amd-suspend-fix
```

```
cat <<'EOF' > /usr/lib/systemd/system-sleep/99-t14-g3-amd-resume-fix
#!/bin/sh -ex

case $1 in
    # This always unblocks wifi, even if it was blocked before, cam be improved
    # later on.
    post) rfkill unblock wifi ;;
esac
EOF
chmod 755 /usr/lib/systemd/system-sleep/99-t14-g3-amd-resume-fix
```

Unfortunately this does not fix the problem. Everything regarding Wifi hangs as
soon as NetworkManager recieves the suspend event and disconnects the Wifi card
which happens before the scripts in system-sleep are run.

Disconnecting Wifi manually before suspending, works fine. With the above hook
scripts, a workaround has been made which is good enough for now.


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

NOTE: This section does not work, so ignore for now. Since we need a very new
Kernel anyways for now without secure boot it is until can sign the Kernel and
modules with our own key.

A Machine Owner Key (MOK), needs to be created and enrolled to Secure Boot. That
 key is then used to sign all extra Kernel modules.

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

