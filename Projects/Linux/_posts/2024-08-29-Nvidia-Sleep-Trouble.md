---
layout: post
author: 248nonny
---

So. School is starting up again soon, and I'll be in my second year at UBC studying Engineering Physics. I was trying to get my laptop school-ready by enabling and experimenting with myriad power management settings, and naturally on of these was to enable power management for my discrete nvidia graphics card, to avoid it chugging my whole battery while I'm taking notes.

## NixOS Options

There are nixos config options which should enable the nvidia driver and power management, so I turned them on, and at first glance everything was fine; the output of "cat /sys/bus/pci/devices/0000:01:00.0/power/runtime_status" was "suspended," showing that my graphics card (the pci device with address 0000:01:00.0) is indeed in sleep mode. However, when I unplug my device, or boot on battery power, something mysterious happened: the card went into "active" mode. nvidia-smi showed that nothing was using the card, and "cat /sys/bus/pci/devices/0000\:01\:00.0/power/control" returned "auto," showing that power management was indeed enabled, so why was the card active?


## The NVIDIA Audio Card

It turns out that the nvidia chip registers as two pci devices, one 3D graphics device and one audio device (presumably for audio out of the HDMI port on my laptop and whatnot). The audio device has a pci address of 00:01:00.1 on my device. I noticed that while connected to my laptop charger, "cat /sys/bus/pci/devices/0000\:01\:00.1/power/control" would also give me "auto," meaning power management was indeed enabled for the audio device. On battery power, however, running that command would give me the error "cat: '/sys/bus/pci/.../power/control' No such file or directory." How odd. Now, even upon reconnecting to the charger, the audio device was nowhere to be found! I tried 'lspci' and a few other commands, but something had removed the audio card from the system.

I experimented with removing the audio device manually (while plugged in), and it seemes that the nvidia card refuses to sleep unless both the graphics and audio devices are present and in sleep mode; this is what I believe causes the card to not sleep on battery power.


## But What's Removing the Card?

I used "udevadm monitor" to listen to kernel level uevents when I unplug my laptop, and lo and behold, something in the kernel decides to nuke the audio device:

```
(insert output here)
```

How strange. It's not even an os level udevrule, since the kernel removes the device first, and udev just reacts to the removed device; that means that the situation can't just be fixed with an extra udev rule.

## Trying to Fix It

I tried all the easy(ish) fixed I could think of; I tried using every version of the nvidia driver I could get my hands on, I tried using other vanilla kernel versions, I tried using the xanmod and zen linux kernels, but all to no avail. I even repartitioned my hard drive to install arch linux to see if the same thing happened, and guess what: it still happened!! Also, no one online has the same issue as me, so it seems pretty clear that the issue is with my laptop, probably at the BIOS level.

However, there was a ray of light: when using the nouveau driver (an open source nvidia driver that's missing many features), both the graphics and the audio nvidia devices are removed on battery power!! At first this may seem like an inconsequential or perhaps bad thing, but I switched back the nvidia driver and checked the kernel logs (with dmesg) and found this: `NVRM: Attempting to remove device 0000:01:00.0 with non-zero usage count!`

So it seems that the nvidia driver somehow blocks the graphics device from being removed, but not the audio device. The reason that this is good news is that something kernel-level (the nvidia driver) is able to prevent the device from being removed, which means that something kernel-level could also preserve the audio device.

## Kernel Patches, yay!

Before this, I've never tried tampering with the linux source code, but this seemed like the perfect opportunity! I decided I'd try to modify the kernel's code to block any attempts at removing the nvidia audio device.

DISCLAIMER: The code I wrote for this and the modifications I made are objectively terrible ideas, both in terms of best practice and in terms of what they do :D copy at your own risk.

Anyways, I started by modifying the `pci_device_remove` function in `linux/drivers/pci/pci-driver.c` by adding an if statement. If the device's pci id and vendor id match the audio cards, do nothing and exit the function, otherwise carry on.

This sadly didn't solve the problem, but it worked as a sort of hello-world to set up the kernel dev environment and to get nixos to compile my patch into the kernel when rebuilding the system.

I then fed this function into ftrace to see which functions call it; ftrace only tells you one level of function calls though, and I wanted to see the whole chain, so I repeatedly used ftrace to figure out which functions call which functions within the kernel when it removes the nvidia audio card. Here's the function tree I ended up with:

<div style="max-height: 300px; overflow-y: scroll;">
{% highlight none %}
pci_device_remove
^
device_release_driver_internal

  ^bus_remove_device
    ^device_del
      ^snd_unregister_device
        ^snd_hwdep_dev_disconnect
          ^(snd_device_disconnect_all)
        ^snd_pcm_dev_disconnect
          ^(snd_device_disconnect_all)
        ^snd_device_disconnect_all
            ^snd_card_disconnect.part.0
              ^snd_card_free
                ^(pci_device_remove)
      ^cdev_device_del
        ^evdev_disconnect
          ^__input_unregister_device
            ^(input_unregister_device)

      ^input_unregister_device
        ^snd_jack_dev_disconnect
          ^(snd_card_disconnect_all)
          ^snd_jack_dev_free
            ^__snd_device_free
              ^snd_device_free_all
                ^release_card_device
                  ^device_release
                    ^kobject_put --- generic kernel object; probably can stop here.

  ^pci_stop_bus_device
    ^pci_stop_and_remove_bus_device
      ^disable_slot
        ^acpiphp_disable_and_eject_slot
          ^acpiphp_hotplug_notify
            ^acpi_device_hotplug
              ^acpi_hotplug_work_fn
                ^process_one_work  --- I think this is where we stop; kernel is just doing anything
                  ^worker_thread x many
                  ^bh_worker x many
{% endhighlight %}
</div>


## The Patch

After much experimentation, I ended up adding an if statement to the `acpi_hotplug_work_fn.` It seems that there is an ACPI call to remove either `device:01` or `device:02` which is what results in the kernel attempting to remove the nvidia card. I wrote a patch which adds an if statement to block attempts to remove these devices. I also added a kernel parameter `allow_nvidia_audio_removal` so I could undo my patch's effects during runtime:

<div style="max-height: 300px; overflow-y: scroll;">
{% highlight c %}
static void acpi_hotplug_work_fn(struct work_struct *work)
{
    struct acpi_hp_work *hpw = container_of(work, struct acpi_hp_work, work);

    // NVIDIA audio patch
    printk(KERN_INFO "acpi_hotplug_work_fn called...\n");
    const char *name = dev_name(&hpw->adev->dev);
    if (
        !allow_nvidia_removal &&
        name &&
        (strcmp(name, "device:01") == 0 || strcmp(name, "device:02") == 0)
    )
    {
        printk(KERN_INFO "blocked acpi attempt to remove nvidia card.\n");
        return;
    }
    // the rest of the function, which removes the device.
    ...
}
{% endhighlight %}
</div>


It took me a while to test different functions with different `if` conditions, and the above is the culmination of many hours of write -> test -> compile -> repeat. Below is the final patch:

<div style="max-height: 300px; overflow-y: scroll;">
{% highlight c %}
diff -rupN linux-vanilla/drivers/acpi/internal.h linux-nvidiapatch/drivers/acpi/internal.h
--- linux-vanilla/drivers/acpi/internal.h	2024-08-28 15:12:34.001708465 -0700
+++ linux-nvidiapatch/drivers/acpi/internal.h	2024-08-30 20:47:20.127635939 -0700
@@ -11,6 +11,8 @@

 #include <linux/idr.h>

+extern bool allow_nvidia_removal;
+
 extern struct acpi_device *acpi_root;

 int early_acpi_osi_init(void);
diff -rupN linux-vanilla/drivers/acpi/osl.c linux-nvidiapatch/drivers/acpi/osl.c
--- linux-vanilla/drivers/acpi/osl.c	2024-08-28 15:12:34.035708325 -0700
+++ linux-nvidiapatch/drivers/acpi/osl.c	2024-08-30 20:33:42.269156091 -0700
@@ -1148,10 +1148,22 @@ struct acpi_hp_work {
 	u32 src;
 };

+bool allow_nvidia_removal = false;
+core_param(allow_nvidia_audio_removal, allow_nvidia_removal, bool, 0644);
+
 static void acpi_hotplug_work_fn(struct work_struct *work)
 {
 	struct acpi_hp_work *hpw = container_of(work, struct acpi_hp_work, work);

+  // NVIDIA audio patch
+  printk(KERN_INFO "acpi_hotplug_work_fn called...\n");
+
+  const char *name = dev_name(&hpw->adev->dev);
+  if (!allow_nvidia_removal && name && (strcmp(name, "device:01") == 0 || strcmp(name, "device:02") == 0)) {
+    printk(KERN_INFO "blocked acpi attempt to remove nvidia card.\n");
+    return;
+  }
+
 	acpi_os_wait_events_complete();
 	acpi_device_hotplug(hpw->adev, hpw->src);
 	kfree(hpw);
diff -rupN linux-vanilla/init/main.c linux-nvidiapatch/init/main.c
--- linux-vanilla/init/main.c	2024-08-28 15:12:40.025683870 -0700
+++ linux-nvidiapatch/init/main.c	2024-08-28 15:21:39.951150105 -0700
@@ -933,6 +933,7 @@ void start_kernel(void)
 	boot_cpu_hotplug_init();

 	pr_notice("Kernel command line: %s\n", saved_command_line);
+  printk(KERN_INFO "This is the kernel with the custom nvidia patch, yay!\n");
 	/* parameters may set static keys */
 	jump_label_init();
 	parse_early_param();
{% endhighlight %}
</div>



In any case, now I'm ready for school! My power usage (and therefore battery life) are one step closer to fully optimized, meaning less time worrying about plugging my laptop in during school.
