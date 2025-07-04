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

Finally, since this device has a built-in Ethernet port, we enable Ethernet for NetworkManager like:

```
cat <<'EOF' > /etc/NetworkManager/conf.d/10-globally-managed-devices.conf
[keyfile]
unmanaged-devices=*,except:type:wifi,except:type:gsm,except:type:cdma,except:type:ethernet
EOF

systemctl restart NetworkManager
```

## Issues

The Hardware platform of the T14 Gen3 AMD is pretty new, so there are some Linux compatibilitiy issues.

I see the following issues (Kernel `5.19.0-76051900-generic #202207312230~1660780566~22.04~9d60db1`):

- The internal microphone volume is very low volume ~~and unusable~~
- ~~My external USB microphone is way to low volume (not usable)~~ -> **seems fixed, see below**
- ~~Suspend does not work with Wifi is connected (no matter if S3 Linux or Windows and Linux is set in BIOS), both "s2idle" and "deep" crashes, hang or do not properly resume in various variants~~ -> **workaround has been made, see below**
- ~~Hotkeys to switch workspaces do not work, especially when using Workspace matrix~~ -> **fixed, see below**
- ~~Fan speed indicator sometimes does show bogus values when fan is off~~ -> **fixed, see below**
- ~~The WWAN 4G modem does not work, no SIM card detected~~ -> **fixed, see below***
- Screen sometimes turns black (`[drm:amdgpu_job_timedout [amdgpu]] *ERROR* ring sdma0 timeout`), GPU driver crashes (happens every couple of days, reason unknown), needs reboot to get functioning again
- ~~HDMI audio output not available in wireplumber with Kernel 6.1~~ **workaround, see below**

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

Others have reported [similar issues](https://bbs.archlinux.org/viewtopic.php?pid=2056452) with the `acp6x` based sound card.

Pop!_OS uses Wireplumber.

After Update to Kernel 6.1 (including backport of ZFS), the microphone works, but volume is still quite low.

### Fixing external USB microphone

Not sure about this one, the problem went away after using alsamixer and sound settings volume control a couple of times.

This problem is now fixed.

### Fixing suspend

Update: There are some hints in the [Lenovo Forum](https://forums.lenovo.com/t5/Other-Linux-Discussions/T14s-G3-AMD-Linux-Sleep/m-p/5172287?page=1#5758208) which need processing. So far it seems that updating the BIOS to latest version fixes most of the `s2idle` and disables `deep` completely as it is not supported in the first place. So the steps below still stand.

By default neither of the sleep modes possible in BIOS worked for me. One part
of the problem seem to be NetworkManager which simply hangs on suspend and makes things fail completely. So for any successful sleep, NetworkManager needs to be
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

Waking up from `s2idle` requires the latest firmware to be installed. I fetched
`linux-firmware-20220913` which is the newest at the time of writing and it
recently got updates for the amdgpu files.

Extract the latest linux-firmware into `/lib/firmware/updates` and update the
initrd with `update-initramfs -c -k all`. Then reboot.

Everything regarding Wifi hangs as soon as NetworkManager receives the suspend
event and disconnects the Wifi card which happens before the scripts in 
system-sleep are run. Thus, as a workaround, the following hooks are added:

```
cat <<'EOF' > /etc/NetworkManager/dispatcher.d/pre-down.d/99-t14-g3-amd-suspend-fix
#!/bin/sh -ex

# This is a hack to workaround hang if the T114 G3 AMD wifi device. This does
# the trick until a proper fix is found by simply disabling Wifi completely if
# the main wifi is about to go down. It has to be in the pre-down stage as the
# down stage is too late and hangs for unknown reason.

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

Disconnecting Wifi manually before suspending, works fine. With the above hook
scripts, a workaround has been made which is good enough for now.

#### Alternative (untested)

As found in the Ach Wiki https://wiki.archlinux.org/title/Lenovo_ThinkPad_T14_(AMD)_Gen_3

```
cat <<'EOF' > /etc/systemd/system/ath11k-suspend.service
[Unit]
Description=Suspend: rmmod ath11k_pci
Before=sleep.target

[Service]
Type=simple
ExecStart=/usr/sbin/rmmod ath11k_pci

[Install]
WantedBy=sleep.target
EOF

cat <<'EOF' > /etc/systemd/system/ath11k-resume.service
[Unit]
Description=Resume: modprobe ath11k_pci
After=suspend.target

[Service]
Type=simple
ExecStart=/usr/sbin/modprobe ath11k_pci

[Install]
WantedBy=suspend.target
EOF

systemctl enable ath11k-suspend.service
systemctl enable ath11k-resume.service
```

### Disable wakeup from sleep on touchpad activity

The system randomnly wakes up from sleep when carried moved. Seems to be related
to the touchpad. Disable wakeup for it like so:

```
cat <<'EOF' > /etc/udev/rules.d/99-t14-g3-disable-touchpad-wakeup.rules
KERNEL=="i2c-ELAN0678:00", SUBSYSTEM=="i2c", ATTR{power/wakeup}="disabled"
EOF
```

### Fixing hotkeys to switch workspaces

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

### Fixing fan speed indicator

Not a big deal, but the fan speed sometimes get reported wrong. It shows 65535
in the vitals Gnome 3 extension while the fan really is off.

It seems that there is a second fan sensor which reports this bogus value.

```
sensors|grep fan
fan1:           0 RPM
fan2:        65535 RPM
```

This problem is fixed after update to Linux Kernel 6.1 (current at 6.1.0-rc5).

### Fixing WWAN 4G modem (Quectel EM05-G)

The Quectel EM05-G needs a unlock script to be added to ModemManager (see [this PR](https://gitlab.freedesktop.org/mobile-broadband/ModemManager/-/merge_requests/902/diffs)). Furthermore, by default the modem uses
the second SIM slot (eSIM?), so no SIM card is detected by default. The following
set of commands fixes all the issues.

```
sudo apt install -y libmbim-utils
curl -s https://gitlab.freedesktop.org/mobile-broadband/ModemManager/-/raw/ead9f1809f6fa0d94723620b66078da6a168716b/data/dispatcher-fcc-unlock/2c7c | sudo tee /usr/share/ModemManager/fcc-unlock.available.d/2c7c
sudo chmod 755 /usr/share/ModemManager/fcc-unlock.available.d/2c7c
sudo ln -s /usr/share/ModemManager/fcc-unlock.available.d/2c7c /etc/ModemManager/fcc-unlock.d/2c7c:030a
sudo systemctl restart ModemManager
sudo mmcli -m 0 --set-primary-sim-slot=1
```

Afterwards, the modem works just fine.

### Fixing GPU crash

This is the full log of the issue.

```
Sep 22 12:30:03 mose4 kernel: [drm:amdgpu_job_timedout [amdgpu]] *ERROR* ring sdma0 timeout, signaled seq=344841, emitted seq=344843
Sep 22 12:30:03 mose4 kernel: [drm:amdgpu_job_timedout [amdgpu]] *ERROR* Process information: process  pid 0 thread  pid 0
Sep 22 12:30:03 mose4 kernel: amdgpu 0000:04:00.0: amdgpu: GPU reset begin!
Sep 22 12:30:03 mose4 kernel: amdgpu 0000:04:00.0: [drm:amdgpu_ring_test_helper [amdgpu]] *ERROR* ring kiq_2.1.0 test failed (-110)
Sep 22 12:30:03 mose4 kernel: [drm:gfx_v10_0_hw_fini [amdgpu]] *ERROR* KGQ disable failed
Sep 22 12:30:04 mose4 kernel: [drm:gfx_v10_0_cp_gfx_enable.isra.0 [amdgpu]] *ERROR* failed to halt cp gfx
Sep 22 12:30:04 mose4 kernel: [drm] free PSP TMR buffer
Sep 22 12:30:04 mose4 kernel: amdgpu 0000:04:00.0: amdgpu: MODE2 reset
Sep 22 12:30:04 mose4 kernel: amdgpu 0000:04:00.0: amdgpu: GPU reset succeeded, trying to resume
Sep 22 12:30:04 mose4 kernel: [drm] PCIE GART of 1024M enabled (table at 0x000000F43FC00000).
Sep 22 12:30:04 mose4 kernel: [drm] VRAM is lost due to GPU reset!
Sep 22 12:30:04 mose4 kernel: [drm] PSP is resuming...
Sep 22 12:30:04 mose4 kernel: [drm] reserve 0xa00000 from 0xf41b000000 for PSP TMR
Sep 22 12:30:04 mose4 kernel: [drm] failed to load ucode GLOBAL_TAP_DELAYS(0x23)
Sep 22 12:30:04 mose4 kernel: [drm] psp gfx command LOAD_IP_FW(0x6) failed and response status is (0xFFFF0010)
Sep 22 12:30:04 mose4 kernel: [drm] failed to load ucode SE0_TAP_DELAYS(0x24)
Sep 22 12:30:04 mose4 kernel: [drm] psp gfx command LOAD_IP_FW(0x6) failed and response status is (0xFFFF0010)
Sep 22 12:30:04 mose4 kernel: [drm] failed to load ucode SE1_TAP_DELAYS(0x25)
Sep 22 12:30:04 mose4 kernel: [drm] psp gfx command LOAD_IP_FW(0x6) failed and response status is (0xFFFF0010)
Sep 22 12:30:04 mose4 kernel: [drm] failed to load ucode SE2_TAP_DELAYS(0x26)
Sep 22 12:30:04 mose4 kernel: [drm] psp gfx command LOAD_IP_FW(0x6) failed and response status is (0xFFFF0010)
Sep 22 12:30:04 mose4 kernel: [drm] failed to load ucode SE3_TAP_DELAYS(0x27)
Sep 22 12:30:04 mose4 kernel: [drm] psp gfx command LOAD_IP_FW(0x6) failed and response status is (0xFFFF0010)
Sep 22 12:30:04 mose4 kernel: amdgpu 0000:04:00.0: amdgpu: RAS: optional ras ta ucode is not available
```

System is still operational, but the screen (both internal and external) is black.

It happens every once in a while - needs more investigation. I am running an external USB-C display most of the time and so far this crash never happend when not using a USB-C display so there might be some connection.

I updated the BIOS/UEFI to the 1.29 as provided by Lenovo in their Drivers & 
Software download section - let's see if that changes anything (update: it didn't). 
I updated the BIOS/UEFI to 1.32 and the problem still appears.

I updated to Kernel 6.1.0-rc5, let's see if that improves things.

This issue is now tracked at https://gitlab.freedesktop.org/drm/amd/-/issues/2220 - no solution as of now. It might improve with Kernel 6.3.

I updated to Kernel 6.2.0-rc4, let's see if that improves things.

### Fixing HDMI audio output with Kernel 6.1

After update to Kernel 6.1, the HDMI audio output was no longer visible in wireplumber. After debugging, it seems ALSA has trouble getting the ucm profiles. Thus an easy workaround is to disable the UCM profiles like so:

```bash
mkdir -p /etc/wireplumber/main.lua.d
cat <<'EOF' > /etc/wireplumber/main.lua.d/51-alsa-fix-t14g3-hdmi.lua
rule = {
  matches = {
    {
      { "device.name", "equals", "alsa_card.pci-0000_04_00.1" },
    },
  },
  apply_properties = {
    -- Disable UCM profile to fix HDMI output no profiles.
    ["api.alsa.use-ucm"] = false,
  },
}
table.insert(alsa_monitor.rules, rule)
EOF

systemctl --user restart wireplumber
```

Afterwards HDMI audio just works fine again.

### Enable fingerprint reader

```
sudo apt install libpam-fprintd
```

Now you can enroll your fingerprints with `fprintd-enroll`. Then you can use
fingerprint instead of password to login/unlock your user account.

### Green artifacts with Kernel 6.12

- https://bbs.archlinux.org/viewtopic.php?id=301280

Some reports say `amdgpu.dcdebugmask=0x10` fixes this - need to test it.


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
