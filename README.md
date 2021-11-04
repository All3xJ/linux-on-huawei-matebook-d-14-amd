# GNU/Linux on MateBook D 14" (AMD Ryzen 5 3500U 2020)
1. [Touchpad](#touchpad)
2. [Power Management](#power)

![](https://www.sceltanotebook.it/images/stories/huawei-matebook-d-14-2020/huawei-matebook-d-14-2020.webp)

<a name="touchpad"></a>
## Touchpad

**There is a bug** that appears intermittently. Touchpad can become lost and unresponsive after wake from suspend.

A workaround is to unload and reload the touchpad kernel module.

```
sudo modprobe -r i2c_hid_acpi i2c_hid && sudo modprobe i2c_hid_acpi i2c_hid
```

<a name="power"></a>
## Power Management

No automated program like tlp, powertop, laptop-mode-tools, auto-cpufreq, etc improves the battery.
After a lot of research and testing, I created a bash script that do some tweaks to put more things possibile in power saving mode.


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

You have to install ryzenadj and zenstates.
