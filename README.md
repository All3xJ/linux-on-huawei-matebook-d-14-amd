# GNU/Linux on MateBook D 14" (AMD Ryzen 5 3500U 2020)
1. [Touchpad](#Touchpad)
2. [Freeze during intensive task](#Freeze)
3. [WiFi disconnects randomly](#Wifi_disconnects)
4. [Power Management](#Power_Management)
    1. [Power Saving](#Power_Saving)
    2. [Performance](#Performance)
5. [Other](#Other)

![](https://www.sceltanotebook.it/images/stories/huawei-matebook-d-14-2020/huawei-matebook-d-14-2020.webp)



<a name="Touchpad"></a>
## Touchpad

There is a bug that appears intermittently. Touchpad can become lost and unresponsive after wake from suspend.
A workaround is to unload and reload the touchpad kernel module.
```
sudo modprobe -r i2c_hid_acpi i2c_hid && sudo modprobe i2c_hid_acpi i2c_hid
```

You can automate the process using the sleep hooks (more infos here: https://wiki.archlinux.org/title/Power_management#Sleep_hooks just look the part of "root-suspend" and "root-resume" services). Basically when the laptop suspends, you have to unload "i2c_hid_acpi" and "i2c_hid", and when it wakes up you have to reload them.

Here is my /etc/systemd/system/root-suspend.service:
```
[Unit]
Description=Local system suspend actions
Before=sleep.target

[Service]
Type=simple
ExecStart=-/usr/bin/modprobe -r i2c_hid_acpi i2c_hid

[Install]
WantedBy=sleep.target
```

and here is /etc/systemd/system/root-resume.service:
```
[Unit]
Description=Local system resume actions
After=suspend.target

[Service]
Type=simple
ExecStart=/usr/bin/modprobe i2c_hid_acpi i2c_hid

[Install]
WantedBy=suspend.target
```

To enable them on boot, just do:
```
sudo systemctl enable --now root-suspend.service && sudo systemctl enable --now root-resume.service
```



<a name="Freeze"></a>
## Freeze during intensive tasks
It happens that if you do intensive tasks (for example you are in a videocall with your webcam on while you are recording the screen and doing other stuff) the laptop starts to lag until if freezes. I noticed that this happens ONLY if it is charging, so I guess that the temperatures go too high due to the charging and so it lags. Maybe the availability of RAM is related too.

Some users reported that editing the file ```/etc/default/grub```, specifically the line ```GRUB_CMDLINE_LINUX_DEFAULT```, adding ```idle=nomwait iommu=pt``` as boot parameters, it solves the issue:
```
GRUB_CMDLINE_LINUX_DEFAULT="idle=nomwait iommu=pt"
```
after you edited the file you have to update the grub, it should work by doing:
```
sudo grub-mkconfig -o /boot/grub/grub.cfg
```

If the issue does not fix, I suggest you:
1) When you do intensive tasks and you notice the lag, you have to unplug the charger. 
2) Increase the swap partition size





<a name="Wifi_disconnects"></a>
## Wifi disconnects randomly

Wifi can disconnect sometimes randomly. It happens because there is a buggy software that doesn't know how to properly change from 2.4GHz to 5GHz and viceversa. The "solution" that I found is to manually set the wifi network we are connected to in 2.4Ghz OR in 5GHz... this way it doesn't switch between 2GHz and 5GHz. This is doable with your network manager in the settings of your DE.



<a name="Power_Management"></a>
## Power Management


<a name="Power_Saving"></a>
#### Power Saving

No automated program like tlp, powertop, laptop-mode-tools, auto-cpufreq, etc improves the battery in this laptop in my experience.
For some reason, Linux Mint has lowest power consumption compared to other distros... however, after some research and testing, I created a bash script that do some tweaks to put more things possibile in power saving mode, and have the best result in terms of power consumption. You have to install ryzenadj and zenstates to run this.
```
#!/bin/bash

sleep 15 &&
echo 5 > /proc/sys/vm/laptop_mode &&
echo 0 > /proc/sys/kernel/nmi_watchdog &&
echo 1 > /sys/module/snd_hda_intel/parameters/power_save &&
echo Y > /sys/module/snd_hda_intel/parameters/power_save_controller &&
#echo 1 > /sys/devices/system/cpu/sched_mc_power_savings &&
#echo ondemand | tee /sys/devices/system/cpu/cpu*/cpufreq/scaling_governor &&
echo powersave | tee /sys/devices/system/cpu/cpu*/cpufreq/scaling_governor &&
echo 1500 > /proc/sys/vm/dirty_writeback_centisecs &&
echo powersave > /sys/module/pcie_aspm/parameters/policy &&
#echo powersupersave > /sys/module/pcie_aspm/parameters/policy
#for i in /sys/bus/usb/devices/*/power/autosuspend; do echo 1 > $i; done &&
#for i in /sys/bus/pci/devices/*/power_level ; do echo 5 > $i ; done 2>/dev/null
ryzenadj -a 15000 -b 30000 -c 20000 --power-saving &&
echo battery > /sys/class/drm/card0/device/power_dpm_state &&
echo manual > /sys/class/drm/card0/device/power_dpm_force_performance_level &&
echo 2 > /sys/class/drm/card0/device/pp_power_profile_mode &&
echo 0 > /sys/devices/system/cpu/cpufreq/boost &&
zenstates --disable -p 0 &&
zenstates --disable -p 1
```

You can create a systemctl service that starts on boot (for example /etc/systemd/system/pwrsavinggg.service):
```
[Unit]
Description=pwrrrrrsavingggggggg

[Service]
Type=simple
RemainAfterExit=yes
ExecStart=/home/user/pwrsaving.sh

[Install]
WantedBy=multi-user.target
```

and then enable it with:
```
sudo systemctl enable --now pwrsavinggg.service
```

<a name="Performance"></a>
#### Performance

If you want to obtain the best performance, here is a bash script to do so:
```
#!/bin/bash

echo 0 > /proc/sys/vm/laptop_mode &&
echo 0 > /sys/module/snd_hda_intel/parameters/power_save &&
echo N > /sys/module/snd_hda_intel/parameters/power_save_controller &&
echo performance | tee /sys/devices/system/cpu/cpu*/cpufreq/scaling_governor &&
echo 1500 > /proc/sys/vm/dirty_writeback_centisecs &&
echo performance > /sys/module/pcie_aspm/parameters/policy &&
ryzenadj -a 45000 -b 45000 -c 45000 --tctl-temp=90 --max-performance &&
echo performance > /sys/class/drm/card0/device/power_dpm_state &&
#echo high > /sys/class/drm/card0/device/power_dpm_force_performance_level &&
echo manual > /sys/class/drm/card0/device/power_dpm_force_performance_level &&
echo 5 > /sys/class/drm/card0/device/pp_power_profile_mode &&
echo 1 > /sys/devices/system/cpu/cpufreq/boost &&
zenstates --enable -p 0 &&
zenstates --enable -p 1
```



<a name="Other"></a>
#### Other
With this laptop I had one more issue: fan.

**Fan**:
Fan can be loud sometimes, and they can't be controlled via software (with Windows neither), so the best you can do is to follow the [Power Saving](#Power_Saving) paragraph.
